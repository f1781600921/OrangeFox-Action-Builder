name: OrangeFox - Build

# Credits to:
# https://github.com/TeamWin
# https://gitlab.com/OrangeFox
# https://github.com/azwhikaru for Recovery Builder Template
# And all Contributors in every repositories I used

on:
  workflow_dispatch:
    inputs:
      MANIFEST_BRANCH:
        description: 'OrangeFox Branch'
        required: true
        default: '12.1'
        type: choice
        options:
        - 12.1
        - 11.0
      DEVICE_TREE:
        description: 'Custom Recovery Tree'
        required: true
        default: 'https://github.com/f1781600921/android_device_IH519Gtest'
      DEVICE_TREE_BRANCH:
        description: 'Custom Recovery Tree Branch'
        required: true
        default: 'main'
      DEVICE_PATH:
        description: 'Specify your device path.'
        required: true
        default: 'device/ecarx/IHU519G'
      DEVICE_NAME:
        description: 'Specify your Device Codename.'
        required: true
        default: 'IHU519G'
      BUILD_TARGET:
        description: 'Specify your Build Target'
        required: true
        default: 'recovery'
        type: choice
        options:
        - boot
        - recovery
        - vendorboot

jobs:
  build:
    name: Build OFRP by ${{ github.actor }}
    runs-on: ubuntu-latest
    if: github.event.repository.owner.id == github.event.sender.id
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    permissions:
      contents: write
    steps:
    - name: Checkout
      uses: actions/checkout@v3
              
    - name: Clean-up
      uses: rokibhasansagar/slimhub_actions@main

    - name: Swap Space
      uses: pierotofy/set-swap-space@master
      with:
        swap-size-gb: 12
      
    - name: Build Environment
      run: |
        sudo apt install aria2 -y
        git clone https://gitlab.com/OrangeFox/misc/scripts
        cd scripts
        sudo bash setup/android_build_env.sh
      
    - name: Set-up Manifest
      if: github.event.inputs.MANIFEST_BRANCH == '11.0' || github.event.inputs.MANIFEST_BRANCH == '12.1'
      run: |
        mkdir -p ${GITHUB_WORKSPACE}/OrangeFox
        cd ${GITHUB_WORKSPACE}/OrangeFox
        git config --global user.name "Carlo Dandan"
        git config --global user.email "jasminecarlo01@gmail.com"
        git clone https://gitlab.com/OrangeFox/sync.git
        cd sync
        ./orangefox_sync.sh --branch ${{ github.event.inputs.MANIFEST_BRANCH }} --path ${GITHUB_WORKSPACE}/OrangeFox/fox_${{ github.event.inputs.MANIFEST_BRANCH }}

    - name: Patching the Source Code
      run: |
        cd ${GITHUB_WORKSPACE}/OrangeFox/fox_${{ github.event.inputs.MANIFEST_BRANCH }}
        sed -i '/<remove-project name="platform\/external\/gflags"  \/>/d' ${GITHUB_WORKSPACE}/OrangeFox/fox_${{ github.event.inputs.MANIFEST_BRANCH }}/.repo/manifests/remove-minimal.xml
        sed -i '/<project path="bootable\/recovery" name="android_bootable_recovery" remote="TeamWin" revision="android-12.1"\/>/d' ${GITHUB_WORKSPACE}/OrangeFox/fox_${{ github.event.inputs.MANIFEST_BRANCH }}/.repo/manifests/twrp-default.xml
        sed -i 's/<project name="android_hardware_qcom_bootctrl" path="hardware\/qcom-caf\/bootctrl" remote="LineageOS" revision="lineage-19.1-caf" \/>/<project name="android_hardware_qcom_bootctrl" path="hardware\/qcom-caf\/bootctrl" remote="LineageOS" revision="lineage-20.0-caf" \/>/g' ${GITHUB_WORKSPACE}/OrangeFox/fox_${{ github.event.inputs.MANIFEST_BRANCH }}/.repo/manifests/twrp-default.xml
        mv ${GITHUB_WORKSPACE}/OrangeFox/fox_${{ github.event.inputs.MANIFEST_BRANCH }}/bootable/recovery ${GITHUB_WORKSPACE}/OrangeFox/fox_${{ github.event.inputs.MANIFEST_BRANCH }}/bootable/recovery-ofrp
        repo sync
        mv ${GITHUB_WORKSPACE}/OrangeFox/fox_${{ github.event.inputs.MANIFEST_BRANCH }}/bootable/recovery-ofrp ${GITHUB_WORKSPACE}/OrangeFox/fox_${{ github.event.inputs.MANIFEST_BRANCH }}/bootable/recovery
        cd ${GITHUB_WORKSPACE}
        git clone https://github.com/ymdzq/scripts of_patch
        cd ${GITHUB_WORKSPACE}/OrangeFox/fox_${{ github.event.inputs.MANIFEST_BRANCH }}/bootable/recovery
        git fetch https://github.com/sekaiacg/twrp_recovery android-12.1 && git cherry-pick 443565c
        git apply ${GITHUB_WORKSPACE}/of_patch/early-translation.patch
        git apply ${GITHUB_WORKSPACE}/of_patch/fastbootd-optimize.patch
        git apply ${GITHUB_WORKSPACE}/of_patch/Backup_Display_Name.patch
        git apply ${GITHUB_WORKSPACE}/of_patch/5743.patch
        git apply ${GITHUB_WORKSPACE}/of_patch/DST.patch
        git apply ${GITHUB_WORKSPACE}/of_patch/settings.patch
        git apply ${GITHUB_WORKSPACE}/of_patch/AVB20.patch
        git apply ${GITHUB_WORKSPACE}/of_patch/Update_Log_File.patch
        git apply ${GITHUB_WORKSPACE}/of_patch/Set_FBE_Status.patch
        git apply ${GITHUB_WORKSPACE}/of_patch/Unmount-data.patch

    - name: Clone Device Tree
      run: |
        cd ${GITHUB_WORKSPACE}/OrangeFox/fox_${{ github.event.inputs.MANIFEST_BRANCH }}
        git clone ${{ github.event.inputs.DEVICE_TREE }} -b ${{ github.event.inputs.DEVICE_TREE_BRANCH }} ./${{ github.event.inputs.DEVICE_PATH }}
        cd ${{ github.event.inputs.DEVICE_PATH }}
        echo "COMMIT_ID=$(git rev-parse HEAD)" >> $GITHUB_ENV

    - name: Check Manifest Branch
      uses: haya14busa/action-cond@v1
      id: fox_branch
      with:
        cond: ${{ github.event.inputs.MANIFEST_BRANCH == '11.0' || github.event.inputs.MANIFEST_BRANCH == '12.1' }}
        if_true: lunch twrp_${{ github.event.inputs.DEVICE_NAME }}-eng && make clean && mka adbd ${{ github.event.inputs.BUILD_TARGET }}image -j$(nproc --all)
        if_false: lunch omni_${{ github.event.inputs.DEVICE_NAME }}-eng && make clean && mka ${{ github.event.inputs.BUILD_TARGET }}image -j$(nproc --all)
      
    - name: Building OrangeFox
      run: |
        cd ${GITHUB_WORKSPACE}/OrangeFox/fox_${{ github.event.inputs.MANIFEST_BRANCH }}
        set +e
        source build/envsetup.sh
        export ALLOW_MISSING_DEPENDENCIES=true
        set -e
        ${{ steps.fox_branch.outputs.value }}

    - name: Set Build Date # For Build Date Info, currently using Asia/Manila
      run: |
        echo "BUILD_DATE=$(TZ=Asia/Manila date +%Y%m%d)" >> $GITHUB_ENV

    - name: Check if Recovery Exist
      run: |
        cd ${GITHUB_WORKSPACE}/OrangeFox/fox_${{ github.event.inputs.MANIFEST_BRANCH }}
        if [ -f out/target/product/${{ github.event.inputs.DEVICE_NAME }}/OrangeFox*.img ]; then
            echo "CHECK_IMG_IS_OK=true" >> $GITHUB_ENV
            echo "MD5_IMG=$(md5sum out/target/product/${{ github.event.inputs.DEVICE_NAME }}/OrangeFox*.img | cut -d ' ' -f 1)" >> $GITHUB_ENV
        else
            echo "Recovery out directory is empty."
        fi
        if [ -f out/target/product/${{ github.event.inputs.DEVICE_NAME }}/OrangeFox*.zip ]; then
            echo "CHECK_ZIP_IS_OK=true" >> $GITHUB_ENV
            echo "MD5_ZIP=$(md5sum out/target/product/${{ github.event.inputs.DEVICE_NAME }}/OrangeFox*.zip | cut -d ' ' -f 1)" >> $GITHUB_ENV
        else
            echo "Recovery out directory is empty."
        fi

    - name: Upload to Release
      if: env.CHECK_IMG_IS_OK == 'true' && env.CHECK_ZIP_IS_OK == 'true'
      uses: softprops/action-gh-release@v1
      with:
        files: |
          OrangeFox/fox_${{ github.event.inputs.MANIFEST_BRANCH }}/out/target/product/${{ github.event.inputs.DEVICE_NAME }}/OrangeFox*.img
          OrangeFox/fox_${{ github.event.inputs.MANIFEST_BRANCH }}/out/target/product/${{ github.event.inputs.DEVICE_NAME }}/OrangeFox*.zip
        name: Unofficial OrangeFox for ${{ github.event.inputs.DEVICE_NAME }} // ${{ env.BUILD_DATE }}
        tag_name: ${{ github.run_id }}
        body: |
          Build: ${{ github.event.inputs.MANIFEST_BRANCH }}
          Device: [Device Tree/Branch](${{ github.event.inputs.DEVICE_TREE }}/tree/${{ github.event.inputs.DEVICE_TREE_BRANCH }})
          Commit: Most recent [commit](${{ github.event.inputs.DEVICE_TREE }}/commit/${{ env.COMMIT_ID }}) during building.
          MD5 (img): ${{ env.MD5_IMG }}
          MD5 (zip): ${{ env.MD5_ZIP }}
