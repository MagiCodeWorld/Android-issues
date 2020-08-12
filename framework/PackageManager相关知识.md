- Android 5.0之前，/data/app和/system/app下存放的是apk文件，5.0则存放以应用名称命名的目录，在目录下存放的才是apk文件和一个lib目录，lib目录存放应用的so文件
- /data/dalvik-cache下保存大部分apk文件和jar包的odex版本，art模式下odex为oat格式文件
- /data/data/<包名>/目录下，数据目录shared_prefs保存应用的设置文件，database保存应用的数据库文件

