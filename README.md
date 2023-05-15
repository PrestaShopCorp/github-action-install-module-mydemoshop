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
      - name: Get label infos
        id: get_label_infos
        run: |
          echo '${{ github.event.workflow_run.pull_requests[0].number }}'
          label=$(gh api /repos/PrestaShopCorp/ps_mbo/pulls/${{ github.event.workflow_run.pull_requests[0].number }} --jq '.labels[].name | select(. | contains("install"))')
          echo $label
          install=$(echo $label | awk -F/ '{print $1}')
          demo=$(echo $label | awk -F/ '{print $2}')
          name=$(echo $label | awk -F/ '{print $3}')
          echo $install
          echo $demo
          echo $name
          echo """label=$(echo $label)""" >> $GITHUB_OUTPUT
          echo """demo=$(echo $demo)""" >> $GITHUB_OUTPUT
          echo """name=$(echo $name)""" >> $GITHUB_OUTPUT
        env:
          GH_TOKEN: ${{ github.token }}

      - name: Get the download url of the latest artifacts uploaded
        if: ${{ contains(steps.get_label_infos.outputs.label, 'install') }}
        id: get_latest_artifact
        run: |
          url=$(gh api /repos/${{ github.repository }}/actions/runs/${{ github.event.workflow_run.id }}/artifacts --jq '.artifacts[0].archive_download_url')
          if [[ -z "$url" ]]; then
            exit 1
          fi
          echo """url=$(echo $url)""" >> $GITHUB_OUTPUT
        env:
          GH_TOKEN: ${{ github.token }}

      - name: Install Prestashop module on MyDemoShop
        if: ${{ contains(steps.get_label_infos.outputs.label, 'install') }}
        uses: PrestaShopCorp/github-action-install-module-mydemoshop@main
        with:
          environment: "demo-${{ steps.get_label_infos.outputs.demo }}"
          shop_name: ${{ steps.get_label_infos.outputs.name }}
          module_name: ps_mbo
          module_zip_url: ${{ steps.get_latest_artifact.outputs.url }}
        env:
          GOOGLE_APIS_CLIENT_ID: ${{ secrets.GOOGLE_APIS_CLIENT_ID }}
          GOOGLE_APIS_CLIENT_SECRET: ${{ secrets.GOOGLE_APIS_CLIENT_SECRET }}
          GOOGLE_APIS_REFRESH_TOKEN: ${{ secrets.GOOGLE_APIS_REFRESH_TOKEN }}
          INSTALL_MODULE_MYDEMOSHOP_URL: ${{ secrets.INSTALL_MODULE_MYDEMOSHOP_URL }}
