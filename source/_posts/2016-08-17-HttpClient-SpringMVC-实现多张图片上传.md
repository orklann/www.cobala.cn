title: HttpClient + SpringMVC 实现多张图片上传
categories: 工具使用
tags: 工具使用
keywords: 多张图片上传
date: 2016-08-17 11:14:42
---

## 客户端代码

```
    /*
	 * @param picPaths 需要上传的文件路径集合
	 * 
	 * @param requestURL 请求的url
	 * 
	 * @return 返回响应的内容
	 */
	public static void uploadFile(String[] picPaths, String requestURL) {

		String boundary = UUID.randomUUID().toString(); // 边界标识 随机生成
		String prefix = "--", end = "\r\n";
		String content_type = "multipart/form-data"; // 内容类型
		String CHARSET = "utf-8"; // 设置编码
		int TIME_OUT = 10 * 10000000; // 超时时间
		try {
			URL url = new URL(requestURL);
			HttpURLConnection conn = (HttpURLConnection) url.openConnection();
			conn.setReadTimeout(TIME_OUT);
			conn.setConnectTimeout(TIME_OUT);
			conn.setDoInput(true); // 允许输入流
			conn.setDoOutput(true); // 允许输出流
			conn.setUseCaches(false); // 不允许使用缓存
			conn.setRequestMethod("POST"); // 请求方式
			conn.setRequestProperty("Charset", "utf-8"); // 设置编码
			conn.setRequestProperty("connection", "keep-alive");
			conn.setRequestProperty("Content-Type", content_type + ";boundary=" + boundary);
			
			/**
			 * 当文件不为空，把文件包装并且上传
			 */
			OutputStream outputSteam = conn.getOutputStream();
			DataOutputStream dos = new DataOutputStream(outputSteam);


			for (int i = 0; i < picPaths.length; i++) {
				File file = new File(picPaths[i]);

				StringBuffer sb = new StringBuffer();
				sb.append(prefix);
				sb.append(boundary);
				sb.append(end);

				/**
				 * 这里重点注意： name里面的值为服务器端需要key 只有这个key 才可以得到对应的文件
				 * filename是文件的名字，包含后缀名的 比如:abc.png
				 */
				sb.append("Content-Disposition: form-data; name=\"" + "multipartFiles" + "\"; filename=\"" + file.getName() + "\"" + end);
				sb.append("Content-Type: application/octet-stream; charset=" + CHARSET + end);
				sb.append(end);
				dos.write(sb.toString().getBytes());

				InputStream is = new FileInputStream(file);
				byte[] bytes = new byte[8192];// 8k
				int len = 0;
				while ((len = is.read(bytes)) != -1) {
					dos.write(bytes, 0, len);
				}
				is.close();
				dos.write(end.getBytes());// 一个文件结束标志
			}
			
			
			byte[] end_data = (prefix + boundary + prefix + end).getBytes();// 结束
																			// http
																			// 流
			
			dos.write(end_data);
			dos.flush();
			
			
			/**
			 * 获取响应码 200=成功 当响应成功，获取响应的流
			 */
			int res = conn.getResponseCode();
			
			if(res == 200) {
			 InputStream is = conn.getInputStream();  
	          int ch;  
	          StringBuffer b = new StringBuffer();  
	          while ((ch = is.read()) != -1) {  
	              b.append((char) ch);  
	          }  
	          String s = b.toString();  
	          
	          
	          System.out.println(s);
			}

		} catch (MalformedURLException e) {
			e.printStackTrace();
		} catch (IOException e) {
			e.printStackTrace();
		}
	}

```

<!--more-->

## 服务器端


```
    @RequestMapping(value="imgUploads",method=RequestMethod.POST)
	@ResponseBody
	public Result upload(@RequestParam(required=true)MultipartFile[] multipartFiles,HttpServletRequest request){
		
		ResultData data = new ResultData();
		
		List<Map<String,Object>> images = new ArrayList<Map<String,Object>>();
		
		try {
			
			for(int i = 0; i < multipartFiles.length; i++) {
				MultipartFile multipartFile = multipartFiles[i];
			
				log.info("文件长度: " + multipartFile.getSize());
				log.info("文件类型: " + multipartFile.getContentType());
				log.info("文件名称: " + multipartFile.getName());
				log.info("文件原名: " + multipartFile.getOriginalFilename());
				
				byte[] bytes = multipartFile.getBytes();
				
				
				String type = FilenameUtils.getExtension(multipartFile.getOriginalFilename());
					
			    ResultMsg msg = uploadService.ftpUploadImage(bytes, type);
			        
			        //上传成功，保存Image数据
			    if (msg.isResult()) {
			        	
		        	Map<String,Object> dataMap = new HashMap<String,Object>();
		        	
		        	dataMap.put("filePath", msg.getMsg());//原图url
					
					//扩展文件信息，可选(如需使用可以客户端获取)
		        	dataMap.put("size", multipartFile.getSize());
		        	dataMap.put("contentType", multipartFile.getContentType());
		        	dataMap.put("originalFilename", multipartFile.getOriginalFilename());
		        	
		        	images.add(dataMap);
				}
				
				
			}
			
		} catch (Exception e) {
			log.error("批量上传图片失败：" + e.getMessage(),e.getMessage());
		}
		
		data.put("msg", "图片上传成功!");
		
		data.put("results", images);
		
		return new Result(data);
	}

```

## 调用示例


```
	@Test
	public void testUpload() {
		
		String[] uploadFiles =  {"C:\\Users\\wang-lei\\Desktop\\image\\1.png",
		                         "C:\\Users\\wang-lei\\Desktop\\image\\150_75_1.png",
		                         "C:\\Users\\wang-lei\\Desktop\\image\\280_500.png"};

		String requestURL = "http://api.local.net/api/common/upload/imgUploads.json";
		
		UploadImagesTestCase2.uploadFile(uploadFiles, requestURL);
		
		
	}
	
```

### 返回数据示例


```
{
  "code": 0,
  "message": "成功",
  "serverTime": 1471402446400,
  "data": {
    "subCode": 0,
    "subMessage": null,
    "results": [
      {
        "filePath": "0fb619de-3816-4b9c-ace6-1c6508795d5d.png",
        "originalFilename": "1.png",
        "contentType": "image/png",
        "size": 249635
      },
      {
        "filePath": "6fcb3131-8901-41ac-87f9-5fa9f453a7f6.png",
        "originalFilename": "150_75_1.png",
        "contentType": "image/png",
        "size": 17747
      },
      {
        "filePath": "a18819c8-fc93-40d8-bbde-72111a5c872b.png",
        "originalFilename": "280_500.png",
        "contentType": "image/png",
        "size": 201269
      }
    ],
    "msg": "图片上传成功!"
  }
}
```

## 总结

众所周知浏览器上传图片时一般都是一个Form表单中要使用<input type='file' name='file' />来选择上传的文件，并上传，代码如下：

```
<html>
    <head>
        <title>upload</title>
        <meta http-equiv="description" content="this is my page">
        <meta http-equiv="content-type" content="text/html; charset=GB18030">
    </head>

    <body>
        <form action="servlet/UploadFile" method="post"
            enctype="multipart/form-data">
            <input type="file" name="file1" id="file1" />
            <input type="file" name="file2" id="file2" />
            <input type="submit" value="上传" />
        </form>
    </body>
</html>
```


从上面的代码可以看出，有两个文件选择框（file1和file2），在上传文件时，<form>标签必须加上enctype="multipart/form-data"，否则浏览器无法将文件内容上传到服务端。

到底这个post请求向服务端发送了什么信息？从浏览器中可以看到以下请求信息：


![image](http://o6skwsce0.bkt.clouddn.com/upload.png)

有了这些信息就可以使用HttpClient 拼接好请求串来模拟浏览器上传图片文件到服务器了，剩下的工作就是接收这些图片的字节流处理。


