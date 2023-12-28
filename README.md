# Repro

## Issue

`nx release` only updates package.json file in the project root directory
and not in the generated `package.json` file in the outputPath of esbuild build task.

## Requirements

- have a local npm registry running on port 4873 (verdaccio)

## Repro issue from main branch

```sh
npx nx release --verbose --yes --dry-run
```

Terminal output:

```sh

#  >  NX   Running release version for project: cli

# cli 🔍 Reading data for package "my-cli" from apps/cli/package.json
# cli 📄 Resolved the current version as 0.0.0 from git tag "cli@0.0.0".
# cli 📄 Resolved the specifier as "minor" using git history and the conventional commits standard.
# cli ✍️  New version 0.1.0 written to apps/cli/package.json

# UPDATE apps/cli/package.json [dry-run]

#     "name": "my-cli",
# -   "version": "0.0.1",
# +   "version": "0.1.0",
#     "dependencies": {},


#  >  NX   Staging changed files with git because --stage-changes was set

# Would stage files in git with the following command, but --dry-run was set:
# git add apps/cli/package.json

# NOTE: The "dryRun" flag means no changes were made.

#  >  NX   Previewing an entry in apps/cli/CHANGELOG.md for cli@0.1.0


# CREATE apps/cli/CHANGELOG.md [dry-run]
# + ## 0.1.0 (2023-12-28)
# +
# +
# + ### 🚀 Features
# +
# + - **cli:** add cli app
# +
# +
# + ### ❤️  Thank You
# +
# + - Thomas Dekiere


#  >  NX   Committing changes with git

# Would stage files in git with the following command, but --dry-run was set:
# git add apps/cli/CHANGELOG.md

# Would commit all previously staged files in git with the following command, but --dry-run was set:
# git commit --message chore(release): publish --message - project: cli 0.1.0

#  >  NX   Tagging commit with git

# Would tag the current commit in git with the following command, but --dry-run was set:
# git tag --annotate cli@0.1.0 --message cli@0.1.0

# NOTE: The "dryRun" flag means no changelogs were actually created.

#  >  NX   Running target nx-release-publish for project cli:

#     - cli
   
#    With additional flags:
#      --dryRun=true

#  ————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————

# > nx run cli:nx-release-publish


# 📦  my-cli@0.0.1
# === Tarball Contents ===

# 116B  README.md   
# 1.1kB index.cjs   
# 114B  package.json
# === Tarball Details ===
# name:          my-cli                                  
# version:       0.0.1                                   
# filename:      my-cli-0.0.1.tgz                        
# package size:  795 B                                   
# unpacked size: 1.3 kB                                  
# shasum:        ac459d53427fe7587cd41551fa1ca9c5dc040bf3
# integrity:     sha512-AJrDDlXzwKdL0[...]BZX74YVvpp+Tw==
# total files:   3                                       
 
# Would publish to http://localhost:4873/ with tag "latest", but [dry-run] was set
```

==> package being published is 0.0.1 while we expected package version 0.1.0 to be published.

## Script to recreate this repository

### Requirements to run the script 

- have jq installed to adjust `nx.json` and `project.json` files
- have verdaccio local npm registry running

## Script

Execute the following to recreate this repository locally.

```sh
npx create-nx-workspace@latest nx-release-repro --preset ts

# open workspace
cd nx-release-repro

# init git
git init

# create publishable js library 'my-cli' using esbuild
npx nx@latest g @nx/js:lib cli \
  --publishable \
  --importPath=my-cli \
  --directory apps/cli \
  --projectNameAndRootFormat as-provided \
  --bundler esbuild \
  --unitTestRunner none \
  --linter none

# Add initial tag
git commit --allow-empty -m 'chore: create empty repo'
git tag cli@0.0.0

# Commit nx workspace with cli application
git add -A
git commit -m 'feat(cli): add cli app'

# Configure nx release
jq '.release = {
    "projects": ["cli"],
    "projectsRelationship": "independent",
    "git": {
      "commit": true,
      "tag": true
    },
    "version": {
      "generatorOptions": {
        "specifierSource": "conventional-commits",
        "currentVersionResolver": "git-tag"
      }
    },
    "changelog": {
      "projectChangelogs": true
    }
}' nx.json > temp.json && mv temp.json nx.json

git add nx.json
git commit -m 'ci: configure nx release'

## Use local verdaccio registry
npm config set --location project registry 'http://localhost:4873/'
git add .npmrc
git commit -m 'build: use local verdaccio registry'

npx nx build cli
npx nx release --verbose --yes --dry-run

#  >  NX   Running release version for project: cli

# cli 🔍 Reading data for package "my-cli" from apps/cli/package.json
# cli 📄 Resolved the current version as 0.1.0 from git tag "cli@0.1.0".
# cli 🚫 No changes were detected using git history and the conventional commits standard.
# cli 🚫 Skipping versioning "my-cli" as no changes were detected.


#  >  NX   No changes detected for changelogs

#    No changes were detected for any changelog files, so no changelog entries will be generated.


#  >  NX   Running target nx-release-publish for project cli:

#     - cli
   
#    With additional flags:
#      --dryRun=true

#  ————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————

# > nx run cli:nx-release-publish

# Waiting for the debugger to disconnect...
# Waiting for the debugger to disconnect...

# 📦  my-cli@0.0.1
# === Tarball Contents ===

# 116B README.md        
# 114B package.json     
# 672B project.json     
# 27B  src/index.ts     
# 50B  src/lib/cli.ts   
# 417B tsconfig.json    
# 231B tsconfig.lib.json
# === Tarball Details ===
# name:          my-cli                                  
# version:       0.0.1                                   
# filename:      my-cli-0.0.1.tgz                        
# package size:  998 B                                   
# unpacked size: 1.6 kB                                  
# shasum:        7bb191ca9b1f1f044fd76df167c9487dd2068b66
# integrity:     sha512-ERx4eXorpC46g[...]/VBC+lguWRdzg==
# total files:   7                                       
 
# Would publish to http://localhost:4873/ with tag "latest", but [dry-run] was set

## ====> Correct version here but it does not contain the actual files from `dist/apps/cli`

# set nx-release-publish packageRoot setting to dist/apps/cli
jq '.targets["nx-release-publish"] = {
          "executor": "@nx/js:release-publish",
          "options": {
            "packageRoot": "dist/apps/cli"
          }
        }' apps/cli/project.json > tmp.json && mv tmp.json apps/cli/project.json


git add apps/cli/project.json
git commit -m 'build: set release-publish.packageRoot to dist/apps/cli'

## ====> Now try to release again

npx nx release --verbose --yes --dry-run

#  >  NX   Running release version for project: cli

# cli 🔍 Reading data for package "my-cli" from apps/cli/package.json
# cli 📄 Resolved the current version as 0.0.0 from git tag "cli@0.0.0".
# cli 📄 Resolved the specifier as "minor" using git history and the conventional commits standard.
# cli ✍️  New version 0.1.0 written to apps/cli/package.json

# UPDATE apps/cli/package.json [dry-run]

#     "name": "my-cli",
# -   "version": "0.0.1",
# +   "version": "0.1.0",
#     "dependencies": {},


#  >  NX   Staging changed files with git because --stage-changes was set

# Would stage files in git with the following command, but --dry-run was set:
# git add apps/cli/package.json

# NOTE: The "dryRun" flag means no changes were made.

#  >  NX   Previewing an entry in apps/cli/CHANGELOG.md for cli@0.1.0


# CREATE apps/cli/CHANGELOG.md [dry-run]
# + ## 0.1.0 (2023-12-28)
# +
# +
# + ### 🚀 Features
# +
# + - **cli:** add cli app
# +
# +
# + ### ❤️  Thank You
# +
# + - Thomas Dekiere


#  >  NX   Committing changes with git

# Would stage files in git with the following command, but --dry-run was set:
# git add apps/cli/CHANGELOG.md

# Would commit all previously staged files in git with the following command, but --dry-run was set:
# git commit --message chore(release): publish --message - project: cli 0.1.0

#  >  NX   Tagging commit with git

# Would tag the current commit in git with the following command, but --dry-run was set:
# git tag --annotate cli@0.1.0 --message cli@0.1.0

# NOTE: The "dryRun" flag means no changelogs were actually created.

#  >  NX   Running target nx-release-publish for project cli:

#     - cli
   
#    With additional flags:
#      --dryRun=true

#  ————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————

# > nx run cli:nx-release-publish


# 📦  my-cli@0.0.1
# === Tarball Contents ===

# 116B  README.md   
# 1.1kB index.cjs   
# 114B  package.json
# === Tarball Details ===
# name:          my-cli                                  
# version:       0.0.1                                   
# filename:      my-cli-0.0.1.tgz                        
# package size:  795 B                                   
# unpacked size: 1.3 kB                                  
# shasum:        ac459d53427fe7587cd41551fa1ca9c5dc040bf3
# integrity:     sha512-AJrDDlXzwKdL0[...]BZX74YVvpp+Tw==
# total files:   3                                       
 
# Would publish to http://localhost:4873/ with tag "latest", but [dry-run] was set

#  ————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————

#  >  NX   Successfully ran target nx-release-publish for project cli



# NOTE: The "dryRun" flag means no projects were actually published.


## ===> Here nx reports to publish version 0.1.0 while the actual version being published is 0.0.1 (taken from dist/apps/cli/package.json)
```




