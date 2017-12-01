node {

    // ビルド対象リポジトリ
    def repositoryURL = 'https://[your_build_target_project_repository].git'

    // ローカルリポジトリディレクトリ名
    def repositoryName = "[your_repository_name]"

    // Unityプロジェクト名
    def projectName = "[your_project_name]"

    // JenkinsワークスペースにチェックアウトされるUnityプロジェクトパス
    def projectPath = "${WORKSPACE}/${repositoryName}/${projectName}"

    // XcodeアーカイブPath
    def archivePath = "${projectPath}/iOS/build/${projectName}.xcarchive"

    // ユーザーが選択したブランチ名
    def buildBranchName


    stage('InputParam') {

        // リモートからブランチの一覧を取得しプルダウンリスト用に整形する
        def branchList = sh(script: "git ls-remote --heads ${repositoryURL} | sed -e 's#.*refs/heads/##'", returnStdout: true)

        // 入力待ち状態で5分経過したらタイムアウトする
        timeout(time:5,unit:"MINUTES")
        {
            def defaultBranch = "develop"

            // デプロイセッティングの入力待ち
            def inputParam = input(
                message: 'ビルドパラメーターをセットしてください',
                ok: 'ビルド開始',
                parameters: [
                    [$class:"ChoiceParameterDefinition",
                        name: 'branchName',
                        choices: branchList,
                        defaultValue: defaultBranch,
                        description: 'ビルド対象のブランチを選択してください'
                    ],
                ]
            )

            buildBranchName = inputParam.branchName
        }
    }

    stage('Git') {
        dir (repositoryName) {
            // ブランチをチェックアウト
            git( branch: buildBranchName, url: repositoryURL)
        }
    }

    stage('Unity Build') {

        // UnityコマンドPath
        def unityPath = "/Applications/Unity/Unity.app/Contents/MacOS/Unity"

        // Unityビルドスクリプトクラス名
        // ※ここはUnity側に設置したバッチビルド用クラスとメソッド名を指定してください
        def executeMethod = "AppBuildBatch.BuildIOSDevelop"

        // ビルドターゲット
        def buildTarget = "ios"

        // UnityのバッチビルドでXcodeプロジェクトをビルドするためのコマンド文字列を作成
        def buildCommand = "${unityPath} -quit -batchmode"
        buildCommand += " -executeMethod ${executeMethod}"
        buildCommand += " -projectPath ${projectPath}"
        buildCommand += " -buildTarget ${buildTarget}"

        // ビルドコマンドを実行
        sh(script: buildCommand)
    }

    stage('xcode build')
    {
        // XcodeプロジェクトPath
        def xcodeprojDir = "${projectPath}/iOS/"
        def xcodeproj = "${xcodeprojDir}Unity-iPhone.xcodeproj"

        // 設定項目（必要に応じて適宜設定してください）
        def bundleID = "xxxxxxxxx"
        def displayName = "xxxxxxxxxxxxx"
        def versionNumber = "xxxxxxxxxxxxx"
        def BuildNumber = "xxxxxxxxx"

        // Plistbuddyで設定項目を書き換える
        sh("/usr/libexec/Plistbuddy -c \"Set CFBundleIdentifier ${bundleID}\" ${xcodeprojDir}Info.plist")
        sh("/usr/libexec/Plistbuddy -c \"Set CFBundleDisplayName ${displayName}\" ${xcodeprojDir}Info.plist")
        sh("/usr/libexec/Plistbuddy -c \"Set CFBundleShortVersionString ${versionNumber}\" ${xcodeprojDir}Info.plist")
        sh("/usr/libexec/Plistbuddy -c \"Set CFBundleVersion ${BuildNumber}\" ${xcodeprojDir}Info.plist")

        // ビルド環境をクリーン
        sh(script: "xcodebuild clean -project ${xcodeproj}")

        // XcodeBuildコマンドでアーカイブ作成
        def developmentTeamName = "xxxxxxxxxxxxxx"
        def provisioningProfilerSpecifier = "xxxxxxxxxxxxxxx"

        sh(script: "xcodebuild archive -project ${xcodeproj} -archivePath ${archivePath} -configuration Develop -scheme Unity-iPhone -sdk iphoneos10.3 CODE_SIGN_IDENTITY=\"iPhone Distribution\" PROVISIONING_PROFILE_SPECIFIER=${provisioningProfilerSpecifier} DEVELOPMENT_TEAM=${developmentTeamName}")
    }

    stage('ipa export')
    {
        // ipaビルド設定ファイルPath
        def exportOptionFile = "${WORKSPACE}/build/buildparams/exportOptions.plist"

        // ipa出力先
        def exportDir = "${WORKSPACE}/export/${BUILD_NUMBER}/"
        def exportFileName = "${projectName}_${BUILD_NUMBER}.ipa"

        // exportディレクトリがなければ作成
        dir ('export') {
            // XcodeBuildコマンドでipa作成
            sh(script: "xcodebuild -exportArchive -archivePath ${archivePath} -exportPath ${exportDir} -exportOptionsPlist ${exportOptionFile}")

            if( fileExists("${BUILD_NUMBER}/Unity-iPhone.ipa") )
            {
                // そのままだと毎回、Unity-iPhone.ipaというファイルが生成されるので、アーカイブ用にリネーム
                sh(script:"mv ${BUILD_NUMBER}/Unity-iPhone.ipa ${BUILD_NUMBER}/${exportFileName}")

                // アーカイブ
                archiveArtifacts(artifacts: "${BUILD_NUMBER}/${exportFileName}", onlyIfSuccessful: true)
            }
        }
    }
}
