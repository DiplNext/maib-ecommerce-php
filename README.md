# Testing MAIB Payment Gateway

## Install Certificate
1) First of all you need certificate from Maib with name and credential like ![image](https://user-images.githubusercontent.com/42368950/185637931-a952ba87-9c3a-420c-b7b4-64892539d9cc.png)

2) For the second you need to generate ```.pem``` certificate from our ```.pfx``` certificate. And enter the certificate password below
```
openssl pkcs12 -in filename.pfx -out cert.pem -nodes
```
> _Note_: change `filename.pfx` with your certificate name in my example the filename is `0149583.pfx`.
3) You need to convert `.pem` certificate to pkcs12 format for using in NSS database. You can do this command
```
openssl pkcs12 -export -in cert.pem -out cert.p12 -name MAIB-CERT
```
> _Note_: change `MAIB-CERT` with certificate name where you need in NSS database
4) Import `.p12` certificate to NSS database using command below
```
pk12util -i cert.p12 -d sql:/etc/pki/nssdb
```
5) If you want to delete the certificate from NSS database you can use the command below
```
certutil -D -d sql:/etc/pki/nssdb -n "MAIB-CERT"
```
> _Note_: `MAIB-CERT` is the for certificate that you want to delete

## Using with PHP
> _Note_: For using http request with php you want to install [libcurl](https://curl.se/) extension
1) After install curl extension you can simply do the request to **MAIB merchant handler**
> _Note_: **MAIB merchant handler** it's a link that could be modified, but as example I will use `https://maib.ecommerce.md:21440/ecomm/MerchantHandler`
2) For the first one you need to do the authorize  request looks like code below:
```
$post = [
      'command' => 'v',
      'amount' => '12350',
      'currency' => '498',
      'description' => 'description',
      'language' => 'en',
      'client_ip_addr' => '192.168.x.x'
    ];

    $addr = curl_init('https://maib.ecommerce.md:21440/ecomm/MerchantHandler');
    curl_setopt($addr, CURLOPT_CERTINFO, true);
    curl_setopt($addr, CURLOPT_RETURNTRANSFER, 1);
    curl_setopt($addr, CURLOPT_SSLCERT,  'MAIB-CERT');
    curl_setopt($addr, CURLOPT_POST, TRUE);
    curl_setopt($addr, CURLOPT_POSTFIELDS, http_build_query($post));
    $response = curl_exec($addr);
    $errors = curl_error($addr);
    curl_close($addr);
```

>_Note_:
> 1) `command v` it's using for authorize the **SMS Transaction** for more information about transaction type and result from request please read the official documentation of **MAIB E-commerce module**
> 2) The `amount` value should be without decimal units. For example, I have `123.50 MDL` in this script I multiplied by 100 to get the number without decimal units.
> 3) The `currency` value must be the **numeric value** of ISO 4217 format. More information you can found on [ISO Format Official Site](https://www.iso.org/iso-4217-currency-codes.html).
> 4) The `description` value is the simple description of this transaction.
> 5) The `language` value is the language that is used by your user on your e-commerce application.
> 6) The `client_ip_addr` value is ip address of your user
> 7) In request options you can see the option like `CURLOPT_CERTINFO` and `CURLOPT_SSLCERT`. Please specify `CURLOPT_CERTINFO` to `true` and `CURLOPT_SSLCERT` the name of your certificate in NSS database.

If you did everything right you give the response the with your `TRANSACTION_ID`. For more information about the response types and response data please read the official documentation of **MAIB E-commerce module**.
3) When you have `TRANSACTION_ID` you want to redirect your user to **MAIB client handler** with url encoded parameter `trans_id` where is your `TRANSACTION_ID`.<br>The request should be looks like `https://maib.ecommerce.md:21443/ecomm/ClientHandler?trans_id=TRANSACTION_ID` <br> The PHP code for doing this looks like:
```
header("Location: https://maib.ecommerce.md:21443/ecomm/ClientHandler?trans_id=TRANSACTION_ID");
```
> _Note_: Note: MAIB client handler it's a link that could be modified, but as example I will use https://maib.ecommerce.md:21443/ecomm/ClientHandler .
4) After you make the redirect your user was redirected to **MAIB Gateway** where he will have to enter his card data. Depending on this you will receive an response to your callback request which you send to your manager on **MAIB**. If the user returns to this callback you receive `TRANSACTION_ID` in response.
5) After that you need the transaction status, for doing that you need `command c` and you can do this with code below:
```
$post = [
      'command' => 'c',
      'trans_id' => TRANSACTION_ID,
      'client_ip_addr' => '192.168.x.x'
    ];

    $addr = curl_init('https://maib.ecommerce.md:21440/ecomm/MerchantHandler');
    curl_setopt($addr, CURLOPT_CERTINFO, true);
    curl_setopt($addr, CURLOPT_RETURNTRANSFER, 1);
    curl_setopt($addr, CURLOPT_SSLCERT, 'MAIB-CERT');
    curl_setopt($addr, CURLOPT_POST, TRUE);
    curl_setopt($addr, CURLOPT_POSTFIELDS, http_build_query($post));
    $response = curl_exec($addr);
    $errors = curl_error($addr);
    curl_close($addr);
```
In response you receive the transaction information like `RESULT`, `RESULT_CODE`, `RRN` and any information about this transaction. Next you need to check `RESULT`, if it's `OK` you can save the transaction in your database, if it's `FAILED` or `DECLINED` that's means the transaction is failed and you need to return `RESULT_CODE` message to client.
If you receive `PENDING` as `RESULT` you need to wait some time like 5 minutes or something like this after that repeat the `command c` request.
If the `RESULT` stays the same you can do the same procedure.
> _Note_: The time how your `TRANSACTION_ID` keep alive is `10 minutes` after that you receive `RESULT` `TIMEOUT`. If you receive that `RESULT` you need to create a new transaction.<br>
> If you need more information about statuses you can check the official MAIB documentation.
> The `RESULT_CODE` and their messages you can get by contacting your manager from MAIB:

6) Next if you want refund your client you need to use `command r`, how can you do this you can see in code below:
```
$post = [
      'command' => 'r',
      'trans_id' => TRANSACTION_ID,
      'amount' => '12350',
    ];

    $addr = curl_init('https://maib.ecommerce.md:21440/ecomm/MerchantHandler');
    curl_setopt($addr, CURLOPT_CERTINFO, true);
    curl_setopt($addr, CURLOPT_RETURNTRANSFER, 1);
    curl_setopt($addr, CURLOPT_SSLCERT,  'MAIB-CERT');
    curl_setopt($addr, CURLOPT_POST, TRUE);
    curl_setopt($addr, CURLOPT_POSTFIELDS, http_build_query($post));
    $response = curl_exec($addr);
    $errors = curl_error($addr);
    curl_close($addr);
```
In response you get the `RESULT`: `OK`, `FAILED` or `REVERSED`, if you get `OK`, that's means the reverse operation is successfully completed,
if you get `REVESED` that means for this `TRANSACTION_ID` you already reversed the client and if you get `FAILED` you get some `RESULT_CODE`. For more information about  `RESULT_CODE` messages please contact your MAIB manager.<br/>

7) And the last command what you need for successfully completed test for **MAIB E-commerce module** is `command b` which means `close business day`.
It's the simple command that must be executed every day at 23:59.
How you can do that in PHP I will show in code below:
```
$post = [
  'command' => 'b',
];

$addr = curl_init('https://maib.ecommerce.md:21440/ecomm/MerchantHandler');
curl_setopt($addr, CURLOPT_VERBOSE, '1');
curl_setopt($addr, CURLOPT_CERTINFO, true);
curl_setopt($addr, CURLOPT_RETURNTRANSFER, 1);
curl_setopt($addr, CURLOPT_SSLCERT, 'MAIB-CERT');
curl_setopt($addr, CURLOPT_POST, TRUE);
curl_setopt($addr, CURLOPT_POSTFIELDS, http_build_query($post));
$response = curl_exec($addr);
$errors = curl_error($addr);
curl_close($addr);
```



Credits: [MAIB](https://www.maib.md/)<br/>
Author: Ispas Eugeniu <[eugenispas15@gmail.com](mailto:eugenispas15@gmail.com)>
