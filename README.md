# Workflow exemple

```yaml
name: Install module on MyDemoShop
on:
  workflow_run:
    workflows:
      - "Install Module on Shop"
    types:
      - completed

jobs:
  install_module:
    name: Install ps_mbo module
    runs-on: ubuntu-latest
    steps:
      - name: Get the download url of the latest artifacts uploaded
        id: get_latest_artifact
        run: |
          url=$(gh api /repos/${{ github.repository }}/actions/runs/${{ github.event.workflow_run.id }}/artifacts --jq '.artifacts[0].archive_download_url')
          echo $url
          echo """url=$(echo $url)""" >> $GITHUB_OUTPUT
        env:
          GH_TOKEN: ${{ github.token }}

      - name: Install Prestashop module on MyDemoShop
        uses: PrestaShopCorp/github-action-install-module-mydemoshop@main
        with:
          environment: demo-cratik
          shop_name: vgarcia-80
          module_name: ps_mbo
          module_zip_url: ${{ steps.get_latest_artifact.outputs.url }}
        env:
          GOOGLE_APIS_CLIENT_ID: ${{ secrets.GOOGLE_APIS_CLIENT_ID }}
          GOOGLE_APIS_CLIENT_SECRET: ${{ secrets.GOOGLE_APIS_CLIENT_SECRET }}
          GOOGLE_APIS_REFRESH_TOKEN: ${{ secrets.GOOGLE_APIS_REFRESH_TOKEN }}
          INSTALL_MODULE_MYDEMOSHOP_URL: ${{ secrets.INSTALL_MODULE_MYDEMOSHOP_URL }}
