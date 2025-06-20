# document-root
ドキュメントを集約して github pages で公開するためのリポジトリ

各リポジトリでは docs 配下にDocusaurus のルールに従ってドキュメントを記述する。開発者は [@ops-frontier/docusarus](https://www.npmjs.com/package/@ops-frontier/docusaurus) を使用してドキュメントをプレビューしながら執筆し、以下の workflow を自リポジトリに加えることで、 Github Pages にリリースできる。

## workflowの設定
```yaml
name: Release Docs

on:
  push:
    branches:
      - main
    paths:
      - docs/**
jobs:
  build-and-release:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Node.js
        uses: actions/setup-node@v3
        with:
          node-version: 18

      - name: Zip Docs
        run: zip -r docs.zip ./docs # Replace ./docs with the actual directory

      - name: Get or Create Latest Release
        id: get_release
        uses: actions/github-script@v7
        with:
          script: |
            const { data: releases } = await github.rest.repos.listReleases({
              owner: context.repo.owner,
              repo: context.repo.repo
            });
            let release = releases.find(r => r.name === "latest");
            if (!release) {
              const created = await github.rest.repos.createRelease({
                owner: context.repo.owner,
                repo: context.repo.repo,
                tag_name: "latest",
                name: "latest",
                draft: false,
                prerelease: false
              });
              release = created.data;
            }
            core.setOutput('upload_url', release.upload_url);

      - name: Upload Docs Zip to Release
        uses: actions/upload-release-asset@v1
        with:
          upload_url: ${{ steps.get_release.outputs.upload_url }}
          asset_path: ./docs.zip
          asset_name: docs.zip
          asset_content_type: application/zip
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Trigger Deploy Workflow
        run: |
          curl -X POST \
            -H "Accept: application/vnd.github.v3+json" \
            -H "Authorization: token ${{ secrets.PAT_FOR_TRIGGER }}" \
            https://api.github.com/repos/${{ github.repository_owner }}/document-root/dispatches \
            -d '{"event_type": "docs-updated", "client_payload": {"repository": "${{ github.repository }}"}}'
```