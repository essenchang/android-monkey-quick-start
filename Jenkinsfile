pipeline {
    agent any

    stages {
        stage('Monkey Test') {
	    environment {
                ANDROID_HOME="${ANDROID_HOME}"
                PATH="${ANDROID_HOME}/emulator:${ANDROID_HOME}/platform-tools:$PATH"
            }
            steps {
                startEmulator()

                // clean workspakce
                deleteDir()

		// checkout repos
                git branch: 'master', url: 'https://github.com/dino-su/android-monkey-quick-start.git'

		// build and deploy
                sh '''
                cd monkey-recorder && ./gradlew -q installDebug && cd -
                cd monkey-test && ./gradlew -q installDebug installDebugAndroidTest && cd -
                '''

                // init artifact directory
                sh 'mkdir -p output/monkey/'

                // start recording
                sh '''
                adb shell am start -n com.kkbox.sqa.recorder/.MainActivity -a android.intent.action.RUN -d START
                adb shell am instrument -w -e class com.kkbox.sqa.monkey.CalculatorTest#start com.kkbox.sqa.monkey.test/android.support.test.runner.AndroidJUnitRunner
                '''

                // run monkey
                sh '''
                adb shell monkey -p com.android.calculator2 -v 20000 > output/monkey/monkey.log
                sleep 10
                '''

                // screenshot
                sh 'adb shell screencap /sdcard/monkey.png'

                // stop recording
                sh 'adb shell am start -n com.kkbox.sqa.recorder/.MainActivity -a android.intent.action.RUN -d STOP'
            }
            post {
                always {
                    // collecting logs
                    sh '''
                    adb bugreport > output/monkey/bugreport.log
                    adb pull /sdcard/monkey.png output/monkey/monkey.png
                    adb pull /sdcard/recorder.mp4 output/monkey/monkey.mp4
                    '''

                    // shutdown Android emulator
                    sh '''
                    ps -A | grep emulator | awk '{print $1}' | xargs kill -9 || true
                    '''

                    sh '''
                    python monkey-to-junit.py output/monkey/monkey.log > output/monkey/monkey.xml
                    '''

                    junit "output/monkey/*.xml"
                    // FIXME: https://issues.jenkins-ci.org/browse/JENKINS-40561
                    // junit testDataPublishers: [[$class: 'TestDataPublisher']], testResults: 'output/monkey/monkey.xml'

                    // archive
                    dir('output/') { archiveArtifacts 'monkey/' }
                }
            }
        }
    }
}


def startEmulator() {
    // reset Android Emulator
    sh 'emulator -wipe-data @Nexus_4_API_23 &'

    // wait for boot up
    sh '''
    bootanim=""
    failcounter=0
    timeout_in_sec=360

    until [[ "$bootanim" =~ "stopped" ]]; do
        bootanim=`$ANDROID_HOME/platform-tools/adb -e shell getprop init.svc.bootanim 2>&1 &`

        if [[ "$bootanim" =~ "device not found" || "$bootanim" =~ "device offline" || "$bootanim" =~ "running" ]]; then
            let "failcounter += 1"
            echo "Waiting for emulator to start"

            if [[ $failcounter -gt timeout_in_sec ]]; then
                echo "Timeout ($timeout_in_sec seconds) reached; failed to start emulator"
                exit 1
            fi
        fi

        sleep 1
    done

    echo "Emulator is ready"
    '''
}
