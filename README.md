# rahulportfolio
Rahul Kumar Singh - Portfolio


#

name: Create Tag from develop (Auto SemVer)

on:
  workflow_dispatch:
    inputs:
      bump:
        description: "Version bump (auto/patch/minor/major)"
        type: choice
        options: [auto, patch, minor, major]
        default: auto
        required: true

permissions:
  contents: write

jobs:
  validate-and-tag:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout develop (full history)
        uses: actions/checkout@v4
        with:
          ref: develop
          fetch-depth: 0

      - name: Fetch all tags
        run: git fetch --tags --force

      - name: Find latest SemVer tag reachable from develop
        id: base
        shell: bash
        run: |
          set -euo pipefail
          if BASE=$(git describe --tags --match '[0-9]*.[0-9]*.[0-9]*' --abbrev=0 2>/dev/null); then
            echo "base=$BASE" >> "$GITHUB_OUTPUT"
          else
            echo "base=0.0.0" >> "$GITHUB_OUTPUT"
          fi

      - name: Decide bump (Conventional Commits) or override
        id: bump
        shell: bash
        run: |
          set -euo pipefail
          INPUT="${{ github.event.inputs.bump }}"
          BASE="${{ steps.base.outputs.base }}"

          if [[ "$BASE" == "0.0.0" ]]; then
            RANGE="$(git rev-list --max-parents=0 HEAD | tail -n1)..HEAD"
          else
            RANGE="$BASE..HEAD"
          fi

          COMMITS="$(git log --pretty=format:'%s%n%b%x00' $RANGE || true)"

          if [[ "$INPUT" != "auto" ]]; then
            CHOICE="$INPUT"
          else
            if [[ -z "$COMMITS" ]]; then
              echo "No new commits since $BASE; refusing duplicate version." >&2
              exit 1
            fi
            if echo "$COMMITS" | tr '\000' '\n' | grep -Eiq '(^|[[:space:]])BREAKING(\ |-)CHANGE(:|!)'; then
              CHOICE=major
            elif echo "$COMMITS" | tr '\000' '\n' | grep -Eq '^[a-zA-Z]+(\(.*\))?!:'; then
              CHOICE=major
            elif echo "$COMMITS" | tr '\000' '\n' | grep -Eq '^feat(\(.*\))?:'; then
              CHOICE=minor
            else
              CHOICE=patch
            fi
          fi

          echo "choice=$CHOICE" >> "$GITHUB_OUTPUT"
          echo "range=$RANGE"   >> "$GITHUB_OUTPUT"

      - name: Compute next SemVer
        id: next
        shell: bash
        run: |
          set -euo pipefail
          BASE="${{ steps.base.outputs.base }}"
          BUMP="${{ steps.bump.outputs.choice }}"
          if [[ ! "$BASE" =~ ^[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
            echo "Base '$BASE' is not SemVer (MAJOR.MINOR.PATCH)"; exit 1
          fi
          IFS='.' read -r MA MI PA <<< "$BASE"
          case "$BUMP" in
            major) MA=$((MA+1)); MI=0; PA=0 ;;
            minor) MI=$((MI+1)); PA=0 ;;
            patch) PA=$((PA+1)) ;;
            *) echo "Unknown bump '$BUMP'"; exit 1 ;;
          esac
          NEXT="${MA}.${MI}.${PA}"
          echo "next=$NEXT" >> "$GITHUB_OUTPUT"

      - name: Configure git author
        run: |
          git config user.name  "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"

      - name: Create and push annotated tag
        shell: bash
        run: |
          TAG="${{ steps.next.outputs.next }}"
          if git rev-parse "refs/tags/$TAG" >/dev/null 2>&1; then
            echo "Tag already exists: $TAG" >&2; exit 1
          fi
          git tag -a "$TAG" -m "Release $TAG"
          git push origin "$TAG"
          echo "Created tag $TAG"

      - name: Summary
        run: |
          echo "📌 Created tag: ${{ steps.next.outputs.next }}"
          echo "🔧 Bump type: ${{ steps.bump.outputs.choice }}"
          echo "🔎 Commit range: ${{ steps.bump.outputs.range }}"

=====

name: Promote Release via PR (DAG)

on:
  workflow_dispatch:
    inputs:
      tag:
        description: "Existing SemVer tag (e.g., 1.2.3)"
        required: true
      target_env:
        description: "Promotion target"
        type: choice
        options: [uat, psup, prod]
        required: true

permissions:
  contents: write
  pull-requests: write

jobs:
  prepare:
    runs-on: ubuntu-latest
    outputs:
      TAG: ${{ steps.out.outputs.TAG }}
      SHA: ${{ steps.out.outputs.SHA }}
      SOURCE: ${{ steps.out.outputs.SOURCE }}
      TARGET: ${{ steps.out.outputs.TARGET }}
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - run: git fetch --all --tags

      - name: Validate SemVer tag & resolve SHA
        id: res
        shell: bash
        run: |
          TAG="${{ github.event.inputs.tag }}"
          if [[ ! "$TAG" =~ ^[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
            echo "Invalid tag format '$TAG'. Use MAJOR.MINOR.PATCH." >&2; exit 1
          fi
          if ! git rev-parse "refs/tags/$TAG" >/dev/null 2>&1; then
            echo "Tag not found: $TAG" >&2; exit 1
          fi
          echo "sha=$(git rev-parse "refs/tags/$TAG^{commit}")" >> "$GITHUB_OUTPUT"
          echo "tag=$TAG" >> "$GITHUB_OUTPUT"

      - name: Map env -> branches
        id: map
        shell: bash
        run: |
          case "${{ github.event.inputs.target_env }}" in
            uat)  echo "src=develop"       >> "$GITHUB_OUTPUT"; echo "tgt=release/uat"  >> "$GITHUB_OUTPUT" ;;
            psup) echo "src=release/uat"   >> "$GITHUB_OUTPUT"; echo "tgt=release/psup" >> "$GITHUB_OUTPUT" ;;
            prod) echo "src=release/psup"  >> "$GITHUB_OUTPUT"; echo "tgt=release/prod" >> "$GITHUB_OUTPUT" ;;
          esac

      - name: Set composite outputs
        id: out
        run: |
          echo "TAG=${{ steps.res.outputs.tag }}"   >> "$GITHUB_OUTPUT"
          echo "SHA=${{ steps.res.outputs.sha }}"   >> "$GITHUB_OUTPUT"
          echo "SOURCE=${{ steps.map.outputs.src }}" >> "$GITHUB_OUTPUT"
          echo "TARGET=${{ steps.map.outputs.tgt }}" >> "$GITHUB_OUTPUT"

  verify-ancestry:
    runs-on: ubuntu-latest
    needs: prepare
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - run: git fetch --all --tags
      - name: Ensure tag commit exists in predecessor branch
        shell: bash
        run: |
          SRC="${{ needs.prepare.outputs.SOURCE }}"
          SHA="${{ needs.prepare.outputs.SHA }}"
          git rev-parse --verify "origin/$SRC" >/dev/null 2>&1 || { echo "Missing branch $SRC"; exit 1; }
          git merge-base --is-ancestor "$SHA" "origin/$SRC" || {
            echo "Tag commit is not in '$SRC'. Promote in order." >&2; exit 1;
          }

  ensure-target:
    runs-on: ubuntu-latest
    needs: prepare
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - run: git fetch --all
      - name: Create target from predecessor if missing
        shell: bash
        run: |
          SRC="${{ needs.prepare.outputs.SOURCE }}"
          TGT="${{ needs.prepare.outputs.TARGET }}"
          if ! git rev-parse --verify "origin/$TGT" >/dev/null 2>&1; then
            echo "Creating $TGT from $SRC"
            git checkout -b "$TGT" "origin/$SRC"
            git push -u origin "$TGT"
          else
            echo "$TGT exists"
          fi

  create-promo-branch:
    runs-on: ubuntu-latest
    needs: [prepare, verify-ancestry, ensure-target]
    outputs:
      BRANCH: ${{ steps.mk.outputs.branch }}
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }

      - name: Configure git author
        run: |
          git config user.name  "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"

      # ✅ Create the promo branch FROM THE TARGET BRANCH (not from the tag)
      - name: Create promo branch from target
        id: mk
        shell: bash
        run: |
          TGT="${{ needs.prepare.outputs.TARGET }}"
          TAG="${{ needs.prepare.outputs.TAG }}"
          SHA="${{ needs.prepare.outputs.SHA }}"
          PROMO_BRANCH="promote/${TAG}-to-${TGT}"

          # Make sure we have the latest remote refs
          git fetch --all --tags

          # Create branch from the target branch tip
          git checkout -B "$PROMO_BRANCH" "origin/$TGT"

          echo "branch=$PROMO_BRANCH" >> "$GITHUB_OUTPUT"

          # Try to merge the tag commit into the promo branch
          set +e
          git merge --no-ff "$SHA" -m "Promote $TAG into $TGT"
          MERGE_STATUS=$?
          set -e

          if [ $MERGE_STATUS -eq 0 ]; then
            echo "Merged tag $TAG into $TGT on $PROMO_BRANCH"
          else
            # If already up-to-date or fast-forward not needed, check if ancestor
            if git merge-base --is-ancestor "$SHA" "$PROMO_BRANCH"; then
              echo "Target already contains $TAG; creating a no-op promotion commit to open PR."
              git commit --allow-empty -m "chore(release): no-op promotion of $TAG into $TGT"
            else
              echo "Merge conflict or unexpected failure while merging $TAG into $TGT."
              echo "Please reconcile manually." >&2
              exit 1
            fi
          fi

          # Push the promo branch
          git push -u origin "$PROMO_BRANCH"

  open-pr:
    runs-on: ubuntu-latest
    needs: [prepare, create-promo-branch]
    permissions:
      contents: read
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Open PR via GitHub API (with PAT)
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.PAT_REPO }}
          script: |
            const owner = context.repo.owner;
            const repo  = context.repo.repo;
            const head  = `${owner}:${{ needs.create-promo-branch.outputs.BRANCH }}`; // owner:branch format
            const base  = `${{ needs.prepare.outputs.TARGET }}`;
            const title = `Promote ${{ needs.prepare.outputs.TAG }} → ${{ needs.prepare.outputs.TARGET }}`;
            const body  = `Promotion PR for **${{ needs.prepare.outputs.TAG }}**.
            Source (predecessor): \`${{ needs.prepare.outputs.SOURCE }}\`
            Target: \`${{ needs.prepare.outputs.TARGET }}\``;

            try {
              const { data: pr } = await github.rest.pulls.create({
                owner, repo, head, base, title, body,
                maintainer_can_modify: true, draft: false
              });
              core.info(`PR created: ${pr.html_url}`);
            } catch (e) {
              if (e.status === 422) {
                // Could be "already exists" or "no diff"; try to find it
                const { data: prs } = await github.rest.pulls.list({
                  owner, repo, state: 'open', head, base
                });
                if (prs.length > 0) {
                  core.info(`Existing PR: ${prs[0].html_url}`);
                } else {
                  core.setFailed(`Could not create or find PR. Details: ${e.message}`);
                }
              } else {
                core.setFailed(`PR create failed: ${e.message}`);
              }
            }


