# 京东云 OpenAPI 签名机制的 Python 实现 #

京东云OpenAPI是将京东云所有资源的管理能力通过API的方式提供出来，供京东云用户和合作伙伴使用。OpenAPI是京东云控制台的有效补充，方便用户更灵活的控制自己的云上资源。

本文档通过step-by-step生成签名以及完整HTTP请求，并将请求发送至京东公共OpenAPI并得到response。

## 前置条件 ##

要使用某个产品线的OpenAPI，首先需要先通过产品文档了解产品功能、计费等方面的信息。

在开始调用京东云OpenAPI之前，需提前在京东云用户中心账户管理下的AccessKey管理页面申请AccessKey和SecretKey密钥对（简称AK/SK）。AK/SK信息请妥善保管，如果遗失可能会造成非法用户使用此信息操作您在云上的资源，给你造成数据和财产损失。AK/SK密钥对允许启用、禁用，启用后可用其调用OpenAPI，禁用后不能用其调用OpenAPI。

京东云OpenAPI使用Restful接口风格，要进行OpenAPI调用需要包含如下信息：请求协议，请求方式，请求地址，请求路径，请求头，请求参数，请求体。

为了您的数据安全，建议务必使用https协议。

## 基本步骤 ##

1. 初始基本配置
2. 生成初始请求header
3. 生成标准请求串
4. 生成待签名字符串
5. 生成签名key
6. 计算签名值
7. 更新请求头
8. 发送请求

## 请求签名具体 Python 实现 ##


### 1. 初始基本配置 ###

	# GET请求
	method = 'GET'
	# 地域
	region = 'cn-north-1'
	# 产品线
	service = 'vm'
    # vm产品线服务地址：{product}.jdcloud-api.com
    full_host = 'vm.jdcloud-api.com'

	# OpenAPI服务的地址和路径格式一般为（默认为https）：
	# https://{product}.jdcloud-api.com/{API版本号}/regions/{地域ID}/{资源名称}/{资源ID(可选)}/{子资源名称(可选)}/{子资源ID(可选)}{:自定义动作(可选)}
    url = 'https://vm.jdcloud-api.com/v1/regions/cn-north-1/instances/i-uvvtdzuxre'

    # 规范请求路径Path：/{apiVersion}/regions/{regionId}/instances/{instanceId}
    canonical_uri = '/v1/regions/cn-north-1/instances/i-uvvtdzuxre'

	# 本例中为GET请求，body为空（若为PUT/POST/PATCH请求，则body为parameter的JSON格式）
	body = ''

### 2. 生成初始请求header ###

	# 当前时间
    now = datetime.datetime.utcfromtimestamp(time.time())

    # 格式化字符串为：'20180812T074253Z'
    jdcloud_date = now.strftime('%Y%m%dT%H%M%SZ')

    # 生成datestamp字符串：“20180812”，用于签名
    datestamp = now.strftime('%Y%m%d')

    # 随机生成的字符串：“58542f21-bda3-4736-9a08-da2339669e52”
    nonce = str(uuid.uuid4())

	# 增加请求header字段：
	# Content-Type：请求数据格式为JSON
	# User-Agent：格式为“JdcloudSdkPython/<SDK版本> <产品线>/<产品线revision>”
	# 请求头如下：
	# 		{'x-jdcloud-nonce': '63e0148e-0fef-4c78-9228-47fefe470b07', 
	# 		'Content-Type': 'application/json', 
	# 		'x-jdcloud-date': '20180812T094103Z', 
	# 		'User-Agent': 'JdcloudSdkPython/1.2.1 vm/1.0.0'}
    headers = {'Content-Type': 'application/json', 'User-Agent': 'JdcloudSdkPython/1.2.1 vm/1.0.0'}
	headers['x-jdcloud-date'] = jdcloud_date
	headers['x-jdcloud-nonce'] = nonce

### 3. 生成标准请求串 ###

	# 生成请求查询字符串，本例为空。
	canonical_querystring = ''

	# 生成CanonicalHeaders字符串，CanonicalHeaders为需要参与签名的请求头及值。要创建规范 HTTP header 列表，请将所有 HTTP header 名称转换为小写，并删除前导空格和尾随空格。通过用字符代码排序HTTP header ，然后遍历 HTTP header 名称来构建规范 HTTP header 列表。使用:分隔名称和值，并添加换行符。
	# 如下：
	# 		content-type:application/json
	# 		host:vm.jdcloud-api.com
	# 		x-jdcloud-date:20180812T074253Z
	# 		x-jdcloud-nonce:58542f21-bda3-4736-9a08-da2339669e52
	#
	canonical_headers = 'content-type' + ':' + headers['Content-Type'] + '\n' + 'host' + ':' + full_host + '\n' + 'x-jdcloud-date' + ':' + headers['x-jdcloud-date'] + '\n' + 'x-jdcloud-nonce' + ':' + headers['x-jdcloud-nonce'] + '\n'

	# SignedHeaders用于告知京东云，请求中的哪些头是签名过程的一部分，京东云可以忽略哪些头（例如，由代理添加的任何附加标头），以验证请求。注意host, x-jdcloud-date, x-jdcloud-nonce必须包含在内。
	signed_headers = 'content-type;host;x-jdcloud-date;x-jdcloud-nonce'
	
	# 生成body的hash（即使body为空字符串）： 
	#		“e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855”
	payload_hash = hashlib.sha256(body.encode('utf-8')).hexdigest()
	
	# 生成标准请求串：
    # GET
    # /v1/regions/cn-north-1/instances/i-uvvtdzuxre
    # 
    # content-type:application/json
    # host:vm.jdcloud-api.com
    # x-jdcloud-date:20180812T074253Z
    # x-jdcloud-nonce:58542f21-bda3-4736-9a08-da2339669e52
    # content-type;host;x-jdcloud-date;x-jdcloud-nonce
	#
    # e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
	canonical_request = (method + '\n' +
                         canonical_uri + '\n' +
                         canonical_querystring + '\n' +
                         canonical_headers + '\n' +
                         signed_headers + '\n' +
                         payload_hash)

### 4. 生成待签名字符串 ###

	# 用于创建请求签名的哈希算法，目前只支持 `JDCLOUD2-HMAC-SHA256`
	algorithm = 'JDCLOUD2-HMAC-SHA256'
	
	# CredentialScope格式为”{时间}/{地域编码}/{产品线}/jdcloud2_request\n”，例如20180130/cn-north-1/vpc/jdcloud2_request\n
	# "20180812/cn-north-1/vm/jdcloud2_request"
	credential_scope = (datestamp + '/' + region + '/' + service + '/' + 'jdcloud2_request')

	# 生成canonical_request的哈希值
	# 		a3349e006302711650240165d89bb9c0b504ff8dc95d665cd98907877fbbd423
	canonical_request_hash = hashlib.sha256(canonical_request.encode('utf-8')).hexdigest()

	# 生成待签名字符串：
	#  StringToSign =
	# 		Algorithm + \n +
	# 		RequestDateTime + \n +
	# 		CredentialScope + \n +
	# 		HashedCanonicalRequest
	# 结果为：
	# JDCLOUD2-HMAC-SHA256
	# 20180812T074253Z
	# 20180812/cn-north-1/vm/jdcloud2_request
	# a3349e006302711650240165d89bb9c0b504ff8dc95d665cd98907877fbbd423

	string_to_sign = algorithm + '\n' + jdcloud_date + '\n' + credential_scope + '\n' + canonical_request_hash

### 5. 生成签名key ###

	# 计算签名key（二进制），其中HMAC(key, data)代表以二进制格式返回输出的HMAC-SHA256函数
	# 伪代码：
	# kSecret = your secret access key
	# kDate = HMAC("JDCLOUD2" + kSecret, Date)
	# kRegion = HMAC(kDate, Region)
	# kService = HMAC(kRegion, Service)
	# kSigning = HMAC(kService, "jdcloud2_request")

	k_date = hmac.new(('JDCLOUD2' + secret_key).encode('utf-8'), datestamp.encode('utf-8'), hashlib.sha256).digest()

	k_region = hmac.new(k_date, region.encode('utf-8'), hashlib.sha256).digest()

	k_service = hmac.new(k_region, service.encode('utf-8'), hashlib.sha256).digest()

	signing_key = hmac.new(k_service, 'jdcloud2_request'.encode('utf-8'), hashlib.sha256).digest()

### 6. 计算签名值 ###

	# 最终生成签名：
	# 	如：9b2026198d3acbf99da395e23a994ed369a0d70f5b4a5d7567dd0caf3009656d
	encoded = string_to_sign.encode('utf-8')
	signature = hmac.new(signing_key, encoded, hashlib.sha256).hexdigest()

### 7. 更新请求头 ###

	# 计算签名后，需要将签名的结果作为Authorization请求头将其添加到请求中
	# 'JDCLOUD2-HMAC-SHA256 Credential=xxxxxxxxxxxxxxxxxxxxxxxxxxx/20180812/cn-north-1/vm/jdcloud2_request, SignedHeaders=content-type;host;x-jdcloud-date;x-jdcloud-nonce, Signature=53305ed8290a26493beec3060d9b1ff7d94cb1a6f2171cd193a1562814c8de37'
	authorization_header = algorithm + ' ' + 'Credential=' + access_key + '/' + credential_scope + ', ' + 'SignedHeaders=' + signed_headers + ', ' + 'Signature=' + signature

	# 更新请求headers，如下：
	#		{'x-jdcloud-date': '20180812T074253Z', 
	#		'x-jdcloud-content-sha256': 'e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855', 
	#		'JDCLOUD2-HMAC-SHA256': 'JDCLOUD2-HMAC-SHA256', 
	#		'x-jdcloud-nonce': '58542f21-bda3-4736-9a08-da2339669e52', 
	#		'User-Agent': 'JdcloudSdkPython/1.2.1 vm/1.0.0', 
	#		'Content-Type': 'application/json', 
	#		'Authorization': 'JDCLOUD2-HMAC-SHA256 Credential=xxxxxxxxxxxxxxxxxxxxxxxxxxx/20180812/cn-north-1/vm/jdcloud2_request, SignedHeaders=content-type;host;x-jdcloud-date;x-jdcloud-nonce, Signature=53305ed8290a26493beec3060d9b1ff7d94cb1a6f2171cd193a1562814c8de37'}

	headers['Authorization'] = authorization_header
	headers['x-jdcloud-date'] = jdcloud_date
	headers['x-jdcloud-content-sha256'] = payload_hash
	headers['JDCLOUD2-HMAC-SHA256'] = 'JDCLOUD2-HMAC-SHA256'
	headers['x-jdcloud-nonce'] = nonce

### 8. 发送请求 ###

#### 8.1 Python 发送请求 ####

	# Python发送请求，并获得相应
	resp = requests.request(method, url, data=body, headers=headers)

#### 8.2 Curl发送请求 ####

请求头中必须包含SignedHeaders中的各个字段：

	# curl -X GET -H "x-jdcloud-date:20180812T094103Z" -H "x-jdcloud-nonce:63e0148e-0fef-4c78-9228-47fefe470b07" -H "Content-Type:application/json" -H "Authorization:JDCLOUD2-HMAC-SHA256 Credential=xxxxxxxxxxxxxxxxxxxxxxxxxxx/20180812/cn-north-1/vm/jdcloud2_request, SignedHeaders=content-type;host;x-jdcloud-date;x-jdcloud-nonce, Signature=53305ed8290a26493beec3060d9b1ff7d94cb1a6f2171cd193a1562814c8de37" https://vm.jdcloud-api.com/v1/regions/cn-north-1/instances/i-uvvtdzuxre
	
## 总结 ##

可将发送OpenAPI请求简单总结为：
1. 生成待签名字符串（包含header各个字段、body的hash等）
2. 生成签名key（包含SK并多层加盐）；
3. 用key签名step 1的待签名字符串，将签名信息加入到header并发送请求（包含AK），服务端按照同样的方式可以校验身份。


## 使用Python SDK发送请求源代码： ##

    from jdcloud_sdk.core.credential import *
    from jdcloud_sdk.services.vm.client.VmClient import *
    from jdcloud_sdk.services.vm.apis.DescribeInstanceRequest import *
    import json
    # 用户AK&SK
    access_key = '<your ak>'
    secret_key = '<your sk>'
    # 地域
    regionId = 'cn-north-1'
	# 要查询的实例ID
    instanceId = 'i-uvvtdzuxre'
    # 生成Credential和Client
    myCredential = Credential(access_key, secret_key)
    myClient = VmClient(myCredential)
    # 定义参数
    myParam = DescribeInstanceParameters(regionId, instanceId)
	# 定义请求
    myRequest = DescribeInstanceRequest(myParam)
    # Client发送请求，并得到响应
    resp = myClient.send(myRequest)
	# 将返回结果以JSON格式打印
    print json.dumps(resp.result, indent=2)
