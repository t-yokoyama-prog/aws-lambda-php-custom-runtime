# README

**本ドキュメントは，`AWS Lambda` でPHPのカスタムランタイムを使用するための前提作業をまとめるものとする.**

[参考]

* [Introducing the new Serverless LAMP stack](https://aws.amazon.com/jp/blogs/compute/introducing-the-new-serverless-lamp-stack/)
* [Creating your custom PHP runtime](https://github.com/aws-samples/php-examples-for-aws-lambda/tree/master/0.1-SimplePhpFunction)

---

## カスタムランタイムの準備

1. `Cloud9` を用いて，ランタイムと依存パッケージのコンパイルに使用する環境を作成する.

     * Cloud9 Environment

        |  Property           |  Value     |
        | ------------------- | ---------- |
        |  Type               |  EC2       |
        |  EC2 instance type  |  t2.micro  |

     * 後続の作業でメモリ不足となる可能性があるため，スワップファイルを作成する.

        ```sh
        # Create a swap file and enable it.
        $ sudo /bin/dd if=/dev/zero of=/swap bs=1M count=1024
        $ sudo chmod 600 /swap
        $ sudo mkswap /swap
        $ sudo swapon /swap
        ```

2. PHPをコンパイルする.

    * `Cloud9` のターミナルで以下のコマンドを実行する.

        ```sh
        # Update packages and install needed compilation dependencies
        $ sudo yum update -y
        
        $ sudo yum install autoconf bison gcc gcc-c++ libcurl-devel libxml2-devel re2c -y

        # Compile OpenSSL v1.0.1 from source, as Amazon Linux uses a newer version than the Lambda Execution Environment, which
        # would otherwise produce an incompatible binary.
        curl -sL http://www.openssl.org/source/openssl-1.0.1k.tar.gz | tar -xvz
        cd openssl-1.0.1k
        ./config && make && sudo make install
        cd ~

        # Download the PHP 8.0.11 source
        mkdir -p ~/environment/php-8-bin
        curl -sL https://github.com/php/php-src/archive/refs/tags/php-8.0.11.tar.gz | tar -xvz
        cd php-src-php-8.0.11

        # Compile PHP 8.0.11 with OpenSSL 1.0.1 support, and install to /home/ec2-user/php-8-bin
        ./buildconf --force
        ./configure --prefix=/home/ec2-user/environment/php-8-bin/ --with-openssl=/usr/local/ssl --with-curl --with-zlib
        make install
        ```

3. `AWS Lambda` の `bootstrap`ファイルと，手順２で作成されたPHPバイナリをzip形式に圧縮する.

    * 圧縮用フォルダを作成する.

        ```sh
        +--runtime
            |-- bootstrap* # ファイルを新規作成
            +-- bin/
                +-- php*   # ~/environment/php-8-bin/bin/php をコピー
        ```

    * `bootstrap`ファイルに，以下のソースコードをコピー＆ペーストする.

        ```php
        #!/opt/bin/php
        <?php
        /* Copyright 2020 Amazon.com, Inc. or its affiliates. All Rights Reserved.
        Permission is hereby granted, free of charge, to any person obtaining a copy of this
        software and associated documentation files (the "Software"), to deal in the Software
        without restriction, including without limitation the rights to use, copy, modify,
        merge, publish, distribute, sublicense, and/or sell copies of the Software, and to
        permit persons to whom the Software is furnished to do so.
        THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED,
        INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A
        PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT
        HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION
        OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE
        SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
        */

        // This invokes Composer's autoloader so that we'll be able to use Guzzle and any other 3rd party libraries we need.
        require __DIR__ . '/vendor/autoload.php';

        // This is the request processing loop. Barring unrecoverable failure, this loop runs until the environment shuts down.
        do {
            // Ask the runtime API for a request to handle.
            $request = getNextRequest();

            // Obtain the function name from the _HANDLER environment variable and ensure the function's code is available.
            $handlerFunction = array_slice(explode('.', $_ENV['_HANDLER']), -1)[0];
            require_once $_ENV['LAMBDA_TASK_ROOT'] . '/src/' . $handlerFunction . '.php';

            // Execute the desired function and obtain the response.
            $response = $handlerFunction($request['payload']);

            // Submit the response back to the runtime API.
            sendResponse($request['invocationId'], $response);
        } while (true);

        function getNextRequest()
        {
            $client = new \GuzzleHttp\Client();
            $response = $client->get('http://' . $_ENV['AWS_LAMBDA_RUNTIME_API'] . '/2018-06-01/runtime/invocation/next');

            return [
            'invocationId' => $response->getHeader('Lambda-Runtime-Aws-Request-Id')[0],
            'payload' => json_decode((string) $response->getBody(), true)
            ];
        }

        function sendResponse($invocationId, $response)
        {
            $client = new \GuzzleHttp\Client();
            $client->post(
            'http://' . $_ENV['AWS_LAMBDA_RUNTIME_API'] . '/2018-06-01/runtime/invocation/' . $invocationId . '/response',
            ['body' => $response]
            );
        }
        ```

    * `Cloud9` のターミナルで以下のコマンドを実行する.

        ```sh
        # Make bootstrap executable.
        $ cd [path to runtime directory]
        $ chmod +x bootstrap

        # Archive files in zip.
        $ zip -r runtime.zip bin bootstrap
        ```

4. 手順３で作成された `runtime.zip` を，`AWS CLI` が実行可能な環境にダウンロードする.  
   後続の手順において，`runtime.zip` から Lambda レイヤーを作成する.

5. `Composer` をインストールする.

    * `Cloud9` のターミナルで以下のコマンドを実行する.

        ```sh
        # Install composer.
        $ cd ~/environment/php-8-bin/
        $ curl -sS https://getcomposer.org/installer | ./bin/php
        ```

6. 依存パッケージ `guzzlehttp/guzzle` をインストールする.

    * `Cloud9` のターミナルで以下のコマンドを実行する.

        ```sh
        # Install Guzzle.
        $ cd ~/environment/php-8-bin/
        $ ./bin/php composer.phar require guzzlehttp/guzzle
        ```

7. 依存パッケージをzip形式に圧縮する.

    * `Cloud9` のターミナルで以下のコマンドを実行する.

        ```sh
        # Archive files in zip.
        $ cd ~/environment/php-8-bin/
        $ zip -r vendor.zip vendor/
        ```

8. 手順７で作成された `vendor.zip` を，`AWS CLI` が実行可能な環境にダウンロードする.  
   後続の手順において，`vendor.zip` から Lambda レイヤーを作成する.

---

## Lambda レイヤーの作成

1. PHPカスタムランタイムのLambda レイヤーを作成する.

    * `AWS CLI` が実行可能な環境で，以下のコマンドを実行する.

        ```sh
        # Example: Case of Tokyo region
        $ aws lambda publish-layer-version \
        >     --layer-name PHP-example-runtime \
        >     --zip-file fileb://runtime.zip \
        >     --region ap-northeast-1
        {
            "Content": {
                "Location": ****,
                "CodeSha256": ****,
                "CodeSize": ****
            },
            "LayerArn": "arn:aws:lambda:ap-northeast-1:****:layer:PHP-example-runtime",
            "LayerVersionArn": "arn:aws:lambda:ap-northeast-1:****:layer:PHP-example-runtime:1",
            "Description": "",
            "CreatedDate": "2021-10-01T00:00:00.000+0000",
            "Version": 1
        }

    * `Lambda` の設定で `LayerVersionArn` を使用するため，コマンドの実行結果を保存する.

2. 依存パッケージのLambda レイヤーを作成する.

    * `AWS CLI` が実行可能な環境で，以下のコマンドを実行する.

        ```sh
        # Example: Case of Tokyo region
        $ aws lambda publish-layer-version \
        >     --layer-name PHP-example-vendor \
        >     --zip-file fileb://vendor.zip \
        >     --region ap-northeast-1
        {
            "Content": {
                "Location": ****,
                "CodeSha256": ****,
                "CodeSize": ****
            },
            "LayerArn": "arn:aws:lambda:ap-northeast-1:****:layer:PHP-example-vendor",
            "LayerVersionArn": "arn:aws:lambda:ap-northeast-1:****:layer:PHP-example-vendor:1",
            "Description": "",
            "CreatedDate": "2021-10-01T00:00:00.000+0000",
            "Version": 1
        }
        ```

    * `Lambda` の設定で `LayerVersionArn` を使用するため，コマンドの実行結果を保存する.

以上
