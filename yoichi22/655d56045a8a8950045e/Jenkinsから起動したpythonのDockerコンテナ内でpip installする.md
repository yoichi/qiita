# 結論

WORKSPACE が書き込み可でマウントされてるのでそこにインストールすればOK

# 【NG】オプションなしで pip install

```groovy:Jenkinsfile
pipeline {
    agent {
        docker { image 'python' }
    }
    stages {
        stage('Test') {
            steps {
                sh 'pip install requests'
                sh 'python -c "import requests; print(requests.get(\"https://qiita.com\").status_code)"'
            }
        }
    }
}
```

→書き込み権限がないのでインストールできない。

```
Installing collected packages: idna, certifi, urllib3, chardet, requests
Could not install packages due to an EnvironmentError: [Errno 13] Permission denied: '/usr/local/lib/python3.7/site-packages/idna-2.7.dist-info'
Consider using the `--user` option or check the permissions.
```

# 【NG】オプション --user を付与

エラーメッセージの提案に従って `pip install --user`

```groovy:Jenkinsfile
pipeline {
    agent {
        docker { image 'python' }
    }
    stages {
        stage('Test') {
            steps {
                sh 'pip install --user requests'
                sh 'python -c "import requests; print(requests.get(\"https://qiita.com\").status_code)"'
            }
        }
    }
}
```

→やっぱり失敗する。HOME が / になっているのでやっぱり権限で引っかかる。

```
Installing collected packages: urllib3, idna, certifi, chardet, requests
Could not install packages due to an EnvironmentError: [Errno 13] Permission denied: '/.local'
Check the permissions.
```

# 【OK】HOME環境変数を設定しオプション --user を付与

```groovy:Jenkinsfile
pipeline {
    agent {
        docker { image 'python' }
    }
    stages {
        stage('Test') {
            steps {
                withEnv(["HOME=${env.WORKSPACE}"]) {
                    sh 'pip install --user requests'
                    sh 'python -c "import requests; print(requests.get(\"https://qiita.com\").status_code)"'
                }
            }
        }
    }
}
```

→ pip install は成功。その後こけてるけど。

```
Installing collected packages: urllib3, certifi, idna, chardet, requests
  The script chardetect is installed in '/var/lib/jenkins/workspace/sandbox_pip-A5NX75H6QOARPETSQHVUDQHUEHONERA5NMQ3VXHJ2ENYHJY7XZEA/.local/bin' which is not on PATH.
  Consider adding this directory to PATH or, if you prefer to suppress this warning, use --no-warn-script-location.
Successfully installed certifi-2018.4.16 chardet-3.0.4 idna-2.7 requests-2.19.1 urllib3-1.23
[Pipeline] sh
[sandbox_pip-A5NX75H6QOARPETSQHVUDQHUEHONERA5NMQ3VXHJ2ENYHJY7XZEA] Running shell script
+ python -c import requests; print(requests.get(https://qiita.com).status_code)
  File "<string>", line 1
    import requests; print(requests.get(https://qiita.com).status_code)
                                             ^
SyntaxError: invalid syntax
```

→エスケープが不足していたのも修正し、ビルド成功するもの：

```groovy:Jenkinsfile
pipeline {
    agent {
        docker { image 'python' }
    }
    stages {
        stage('Test') {
            steps {
                withEnv(["HOME=${env.WORKSPACE}"]) {
                    sh 'pip install --user requests'
                    sh 'python -c "import requests; print(requests.get(\\\"https://qiita.com\\\").status_code)"'
                }
            }
        }
    }
}
```

# 使いみち

* setup.py で制御が閉じる(setup_requiresなどで依存パッケージを指定する)場合は .egg に依存パッケージがインストールされるので何も考えなくても WORKSPACE 以下の書き込みになり困らない
* Dockerfile を用意してイメージをビルドするときに pip install してしまう場合も困らない
* setup.py を書いたり、docker イメージビルドしたりせずにさくっとパッケージ入れて何かしたいときには使えるかも

# 参考文献

* https://jenkins.io/doc/book/pipeline/docker/
* https://stackoverflow.com/questions/50606352/pip-install-not-working-with-jenkins
