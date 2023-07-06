# GitHub Action

This GitHub action is going to take a GitHub artifact, a zip type and deploy it inside a mydemoshop container.
This action is only usable with a workflow run when the workflow jobs where the artifact is generated is finished.
FYI, an artifacts is only available go get via API when the workflow job is over that why we are using a workflow run.

Your artifact needs to be generated as a ZIP type. You can use the Github action [upload-artifact](https://github.com/actions/upload-artifact) to create your zip.

Once your zip is created you can implement the workflow like this inside your repository.

## Workflow usage

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
```

This secret are herited from the PrestashopCorp organization on GitHub:
```yaml
env:
  GOOGLE_APIS_CLIENT_ID: ${{ secrets.GOOGLE_APIS_CLIENT_ID }}
  GOOGLE_APIS_CLIENT_SECRET: ${{ secrets.GOOGLE_APIS_CLIENT_SECRET }}
  GOOGLE_APIS_REFRESH_TOKEN: ${{ secrets.GOOGLE_APIS_REFRESH_TOKEN }}
  INSTALL_MODULE_MYDEMOSHOP_URL: ${{ secrets.INSTALL_MODULE_MYDEMOSHOP_URL }}
```

Once this workflow It's in your repository, create your [mydemoshop](https://www.notion.so/MyDemoShop-Family-Release-be6469393793402fb7ba83911a480975) instance. You will receive a private message on slack that will informed you that your shop is ready. Keep this message information are usefull to create the trigger label.

To trigger the workflow you will need to add a label on your PR. The label needs to be construct like this:
`install/<demo-name>/<shop-name>`

O3T Production send you this message:
```
:drum_with_drumsticks: Hello,
This message is just to inform you that your shop testing2 has been submitted to demo-niak. You should receive soon a second message when your PrestaShop is going to be ready
```

Demo name is `niak` and shop-name is `testing2` (the name that you choose)

Put this label on you PR and the workflow will reconize it to know where to install the module.


That's all folks !

## Error when unzipping

If the workflow end by saying "Error when unzipping module files" see this open issue :
- https://github.com/actions/upload-artifact/issues/39


# Action steps

This is an explaination of every steps:
```
- name: OAuth token
  id: get_oauth_token
  shell: bash
  run: |
    access_token=$( \
    curl --header "Content-Type: application/x-www-form-urlencoded" \
          --data-urlencode "client_secret=${{ env.GOOGLE_APIS_CLIENT_SECRET }}" \
          --data-urlencode "client_id=${{ env.GOOGLE_APIS_CLIENT_ID }}" \
          --data-urlencode "grant_type=refresh_token" \
          --data-urlencode "refresh_token=${{ env.GOOGLE_APIS_REFRESH_TOKEN }}" \
          --location "https://oauth2.googleapis.com/token" | \
    jq -r '.access_token')
    echo """access_token=$access_token""" >> $GITHUB_OUTPUT
```
This first steps will get an Oauth token from a Google Account provide by infra team, the secret send to the Github action  will be used to authorize you to retrieve this token.


```
- name: Install module
  id: install_module
  shell: bash
  run: |
    echo '{"target_environment":"${{ inputs.environment }}","shopname":"${{ inputs.shop_name }}","module_name":"${{ inputs.module_name }}","url":"${{ inputs.module_zip_url }}","http_headers":{"Accept": "application/vnd.github+json","Authorization":"Bearer ${{ github.token }}","X-GitHub-Api-Version":"2022-11-28"}}' > req.json
    install_response=$(curl --header 'Authorization: Bearer ${{ steps.get_oauth_token.outputs.access_token  }}' --header 'Content-Type: application/json' --data @req.json --location '${{ env.INSTALL_MODULE_MYDEMOSHOP_URL }}')
    status=$(echo $install_response | jq -r '.status')
    message=$(echo $install_response | jq -r '.message')
    echo """status=$status""" >> $GITHUB_OUTPUT
    echo """message=$message""" >> $GITHUB_OUTPUT
    echo $message
    if [[ "$status" == "false" ]]; then
      exit 1
    fi
```

This second one will create the json payload depending on you shop information and send it directly to the [mydemoshop api](https://www.notion.so/MyDemoShop-API-and-backstage-1d3846b1a7f04a6abcbe8cda9ee06846) working on appscript.
