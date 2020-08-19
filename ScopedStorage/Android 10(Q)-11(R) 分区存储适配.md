

# Android 10(Q)/11(R) 分区存储适配

*大部分应用都会请求 ( READ_EXTERNAL_STORAGE ) ( WRITE_EXTERNAL_STORAGE ) 存储权限，来做一些诸如在 SD 卡中存储文件或者读取多媒体文件等常规操作。这些应用可能会在磁盘中存储大量文件，即使应用被卸载了还会依然存在。另外，这些应用还可能会读取其他应用的一些敏感文件数据。

为此，Google 终于下定决心在 Android 10 中引入了分区存储，对权限进行场景的细分，按需索取，并在 Android 11 中进行了进一步的调整。

## Android 存储分区情况

Android 中存储可以分为两大类：私有存储和共享存储

* 私有存储 (Private Storage) : 每个应用在都拥有自己的私有目录，其它应用看不到，彼此也无法访问到该目录：
  * 内部存储私有目录` (/data/data/packageName)` ；
  * 外部存储私有目录 `(/sdcard/Android/data/packageName)`，
* 共享存储 (Shared Storage) : 存储其他应用可访问文件， 包含媒体文件、文档文件以及其他文件，对应设备DCIM、Pictures、Alarms、Music、Notifications、Podcasts、Ringtones、Movies、Download等目录。



## Android 10(Q) ：



Android 10 中主要对`共享目录`进行了权限详细的划分，不再能通过绝对路径访问。

受影响的接口：

![scpoed-storage-1](/Users/jinpengxie/Documents/Android/XLearnNotes/ScopedStorage/scpoed-storage-1.png)



### 访问不同分区的方式：

1. 私有目录：和以前的版本一致，可通过 `File()` API 访问，无需申请权限。
2. 共享目录：需要通过`MediaStore`和`Storage Access Framework` API 访问，视具体情况申请权限，下面详细介绍。

其中，对共享目录的权限进行了细分：

1. 无需申请权限的操作：  
   通过 `MediaStore API`对媒体集、文件集进行媒体/文件的添加、对 **自身APP** 创建的 媒体/文件 进行查询、修改、删除的操作。

2. 需要申请`READ_EXTERNAL_STORAGE `权限：  
   通过 `MediaStore API`对所有的媒体集进行查询、修改、删除的操作。

3. 调用 `Storage Access Framework API` ：  
   会启动系统的文件选择器向用户申请操作指定的文件

   

新的访问方式：

![scpoed-storage-2](/Users/jinpengxie/Documents/Android/XLearnNotes/ScopedStorage/scoped-storage-2.png)



## Android 11 (R):

Android 11 (R) 在 Android 10 (Q) 中分区存储的基础上进行了调整



### 1. 新增执行批量操作

> 为实现各种设备之间的一致性并增加用户便利性，Android 11 向 MediaStore API 中添加了多种方法。对于希望简化特定媒体文件更改流程（例如在原位置编辑照片）的应用而言，这些方法尤为有用。

MediaStore API 新增的方法

| 方法                                                         | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| createWriteRequest (ContentResolver, Collection)             | 用户向应用授予对指定媒体文件组的写入访问权限的请求。         |
| createFavoriteRequest (ContentResolver, Collection, boolean) | 用户将设备上指定的媒体文件标记为 “收藏” 的请求。对该文件具有读取访问权限的任何应用都可以看到用户已将该文件标记为 “收藏”。 |
| createTrashRequest (ContentResolver, Collection, boolean)    | 用户将指定的媒体文件放入设备垃圾箱的请求。垃圾箱中的内容在特定时间段（默认为 7 天）后会永久删除。 |
| createDeleteRequest (ContentResolver, Collection)            | 用户立即永久删除指定的媒体文件（而不是先将其放入垃圾箱）的请求。 |

系统在调用以上任何一个方法后，会构建一个 PendingIntent 对象。应用调用此 intent 后，用户会看到一个对话框，请求用户同意应用更新或删除指定的媒体文件。



### 2. 使用直接文件路径和原生库访问文件

>为了帮助您的应用更顺畅地使用第三方媒体库，Android 11 允许您使用除 MediaStore API 之外的 API 访问共享存储空间中的媒体文件。不过，您也可以转而选择使用以下任一 API 直接访问媒体文件：
>
>File API。  
>原生库，例如 fopen()。

简单来说就是，可以通过 `File()` 等API 访问有权限访问的媒体集了。

##### 性能：

通过 `File ()` 等直接通过路径访问的 API 实际上也会映射为`MediaStore` API 。  
按文件路径顺序读取的时候性能相当；随机读取和写入的时候则会更慢，所以还是推荐直接使用 `MediaStore`API。 

### 3. 新增权限

`MANAGE_EXTERNAL_STORAGE` :
 类似以前的 `READ_EXTERNAL_STORAGE ` + `WRITE_EXTERNAL_STORAGE `，除了应用专有目录都可以访问。

 应用可通过执行以下操作向用户请求名为所有文件访问权限的特殊应用访问权限：

1. 在清单中声明 `MANAGE_EXTERNAL_STORAGE` 权限。  
2. 使用 `ACTION_MANAGE_ALL_FILES_ACCESS_PERMISSION` intent 操作将用户引导至一个系统设置页面，在该页面上，用户可以为您的应用启用以下选项：授予所有文件的管理权限。

* 在 Google Play 上架的话，需要提交使用此权限的说明，只有指定的几种类型的 APP 才能使用。



## Sample

* 使用 `MediaStore` 增删改查媒体集

* 使用 `Storage Access Framework` 访问文件集

  

### 1. 媒体集


#### 1） 查询媒体集（需要 READ_EXTERNAL_STORAGE 权限）

实际上 `MediaStore ` 是以前就有的 API ，不同的是过去主要通过 `MediaStore.Video.Media._DATA` 这个 colum 请求原始数据，可以得到绝对`Uri` ，现在需要请求`MediaStore.Video.Media._ID`来得到相对`Uri`再进行处理。

```kotlin
// Need the READ_EXTERNAL_STORAGE permission if accessing video files that your
// app didn't create.

// Container for information about each video.
data class Video(
    val uri: Uri,
    val name: String,
    val duration: Int,
    val size: Int
)
val videoList = mutableListOf<Video>()

val projection = arrayOf(
    MediaStore.Video.Media._ID,
    MediaStore.Video.Media.DISPLAY_NAME,
    MediaStore.Video.Media.DURATION,
    MediaStore.Video.Media.SIZE
)

// Show only videos that are at least 5 minutes in duration.
val selection = "${MediaStore.Video.Media.DURATION} >= ?"
val selectionArgs = arrayOf(
    TimeUnit.MILLISECONDS.convert(5, TimeUnit.MINUTES).toString()
)

// Display videos in alphabetical order based on their display name.
val sortOrder = "${MediaStore.Video.Media.DISPLAY_NAME} ASC"

val query = ContentResolver.query(
    MediaStore.Video.Media.EXTERNAL_CONTENT_URI,
    projection,
    selection,
    selectionArgs,
    sortOrder
)
query?.use { cursor ->
    // Cache column indices.
    val idColumn = cursor.getColumnIndexOrThrow(MediaStore.Video.Media._ID)
    val nameColumn =
            cursor.getColumnIndexOrThrow(MediaStore.Video.Media.DISPLAY_NAME)
    val durationColumn =
            cursor.getColumnIndexOrThrow(MediaStore.Video.Media.DURATION)
    val sizeColumn = cursor.getColumnIndexOrThrow(MediaStore.Video.Media.SIZE)

    while (cursor.moveToNext()) {
        // Get values of columns for a given video.
        val id = cursor.getLong(idColumn)
        val name = cursor.getString(nameColumn)
        val duration = cursor.getInt(durationColumn)
        val size = cursor.getInt(sizeColumn)

        val contentUri: Uri = ContentUris.withAppendedId(
            MediaStore.Video.Media.EXTERNAL_CONTENT_URI,
            id
        )

        // Stores column values and the contentUri in a local object
        // that represents the media file.
        videoList += Video(contentUri, name, duration, size)
    }
}

```



#### 2）插入媒体集（无需权限）

```kotlin
// Add a media item that other apps shouldn't see until the item is
// fully written to the media store.
val resolver = applicationContext.contentResolver

// Find all audio files on the primary external storage device.
// On API <= 28, use VOLUME_EXTERNAL instead.
val audioCollection = MediaStore.Audio.Media
        .getContentUri(MediaStore.VOLUME_EXTERNAL_PRIMARY)

val songDetails = ContentValues().apply {
    put(MediaStore.Audio.Media.DISPLAY_NAME, "My Workout Playlist.mp3")
    put(MediaStore.Audio.Media.IS_PENDING, 1)
}

val songContentUri = resolver.insert(audioCollection, songDetails)

resolver.openFileDescriptor(songContentUri, "w", null).use { pfd ->
    // Write data into the pending audio file.
}

// Now that we're finished, release the "pending" status, and allow other apps
// to play the audio track.
songDetails.clear()
songDetails.put(MediaStore.Audio.Media.IS_PENDING, 0)
resolver.update(songContentUri, songDetails, null, null)
```



#### 3）更新自己创建的媒体集（无需权限）

删除类似

```kotlin
// Updates an existing media item.
val mediaId = // MediaStore.Audio.Media._ID of item to update.
val resolver = applicationContext.contentResolver

// When performing a single item update, prefer using the ID
val selection = "${MediaStore.Audio.Media._ID} = ?"

// By using selection + args we protect against improper escaping of // values.
val selectionArgs = arrayOf(mediaId.toString())

// Update an existing song.
val updatedSongDetails = ContentValues().apply {
    put(MediaStore.Audio.Media.DISPLAY_NAME, "My Favorite Song.mp3")
}

// Use the individual song's URI to represent the collection that's
// updated.
val numSongsUpdated = resolver.update(
        myFavoriteSongUri,
        updatedSongDetails,
        selection,
        selectionArgs)
```



#### 4）更新/删除其它媒体创建的媒体集

若已经开启分区存储则会抛出 `RecoverableSecurityException`，捕获并通过`SAF`请求权限



```kotlin
// Apply a grayscale filter to the image at the given content URI.
try {
    contentResolver.openFileDescriptor(image-content-uri, "w")?.use {
        setGrayscaleFilter(it)
    }
} catch (securityException: SecurityException) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
        val recoverableSecurityException = securityException as?
            RecoverableSecurityException ?:
            throw RuntimeException(securityException.message, securityException)

        val intentSender =
            recoverableSecurityException.userAction.actionIntent.intentSender
        intentSender?.let {
            startIntentSenderForResult(intentSender, image-request-code,
                    null, 0, 0, 0, null)
        }
    } else {
        throw RuntimeException(securityException.message, securityException)
    }
}

```

### 2. 文件集 （通过 SAF）



#### 1）创建文档

注：创建操作若重名的话不会覆盖原文档，会添加 (1) 最为后缀，如 document.pdf -> document(1).pdf

```kotlin
// Request code for creating a PDF document.
const val CREATE_FILE = 1

private fun createFile(pickerInitialUri: Uri) {
    val intent = Intent(Intent.ACTION_CREATE_DOCUMENT).apply {
        addCategory(Intent.CATEGORY_OPENABLE)
        type = "application/pdf"
        putExtra(Intent.EXTRA_TITLE, "invoice.pdf")

        // Optionally, specify a URI for the directory that should be opened in
        // the system file picker before your app creates the document.
        putExtra(DocumentsContract.EXTRA_INITIAL_URI, pickerInitialUri)
    }
    startActivityForResult(intent, CREATE_FILE)
}
```



#### 2）打开文档

建议使用 type 设置 MIME 类型

```kotlin
// Request code for selecting a PDF document.
const val PICK_PDF_FILE = 2

fun openFile(pickerInitialUri: uri) {
    val intent = Intent(Intent.ACTION_OPEN_DOCUMENT).apply {
        addCategory(Intent.CATEGORY_OPENABLE)
        type = "application/pdf"

        // Optionally, specify a URI for the file that should appear in the
        // system file picker when it loads.
        putExtra(DocumentsContract.EXTRA_INITIAL_URI, pickerInitialUri)
    }

    startActivityForResult(intent, PICK_PDF_FILE)
}
```



#### 3）授予对目录内容的访问权限

用户选择目录后，可访问该目录下的所有内容

***Android 11 中无法访问 Downloads***

```kotlin
fun openDirectory(pickerInitialUri: Uri) {
    // Choose a directory using the system's file picker.
    val intent = Intent(Intent.ACTION_OPEN_DOCUMENT_TREE).apply {
        // Provide read access to files and sub-directories in the user-selected
        // directory.
        flags = Intent.FLAG_GRANT_READ_URI_PERMISSION

        // Optionally, specify a URI for the directory that should be opened in
        // the system file picker when it loads.
        putExtra(DocumentsContract.EXTRA_INITIAL_URI, pickerInitialUri)
    }

    startActivityForResult(intent, your-request-code)
}
```



#### 4）永久获取目录访问权限

上面提到的授权是临时性的，重启后则会失效。可以通过下面的方法获取相应目录永久性的权限

```kotlin
val contentResolver = applicationContext.contentResolver

val takeFlags: Int = Intent.FLAG_GRANT_READ_URI_PERMISSION or
        Intent.FLAG_GRANT_WRITE_URI_PERMISSION
// Check for the freshest data.
contentResolver.takePersistableUriPermission(uri, takeFlags)
```



#### 5）SAF API 响应

`SAF API` 调用后都是通过 `onActivityResult`来相应动作

```kotlin
override fun onActivityResult(
        requestCode: Int, resultCode: Int, resultData: Intent?) {
    if (requestCode == your-request-code
            && resultCode == Activity.RESULT_OK) {
        // The result data contains a URI for the document or directory that
        // the user selected.
        resultData?.data?.also { uri ->
            // Perform operations on the document using its URI.
        }
    }
}
```

#### 6) 其它操作

除了上面的操作之外，对文档其它的复制、移动等操作都是通过设置不同的 FLAG 来实现，见 [`Document.COLUMN_FLAGS`](https://developer.android.com/reference/android/provider/DocumentsContract.Document#COLUMN_FLAGS)

#### 3. 批量操作媒体集



构建一个媒体集的写入操作 `createWriteRequest()`

```kotlin
val urisToModify = /* A collection of content URIs to modify. */
val editPendingIntent = MediaStore.createWriteRequest(contentResolver,
        urisToModify)

// Launch a system prompt requesting user permission for the operation.
startIntentSenderForResult(editPendingIntent.intentSender, EDIT_REQUEST_CODE,
    null, 0, 0, 0)

//相应
override fun onActivityResult(requestCode: Int, resultCode: Int,
                 data: Intent?) {
    ...
    when (requestCode) {
        EDIT_REQUEST_CODE ->
            if (resultCode == Activity.RESULT_OK) {
                /* Edit request granted; proceed. */
            } else {
                /* Edit request not granted; explain to the user. */
            }
    }
}
```

`createFavoriteRequest()` `createTrashRequest()` `createDeleteRequest()` 同理


![批量删除图片](/Users/jinpengxie/Documents/Android/XLearnNotes/ScopedStorage/scoped-storage-5.png)





### 适配和兼容

在 targetSDK = 29 APP 中，在 `AndroidManifes` 设置 `requestLegacyExternalStorage="true"` 启用兼容模式，以传统分区模式运行。

```xml
   <manifest ... >
      <!-- This attribute is "false" by default on apps targeting
           Android 10 or higher. -->
      <application android:requestLegacyExternalStorage="true" ... >
        ...
      </application>
    </manifest>
```



>**注意**：如果某个应用在安装时启用了传统外部存储，则该应用会保持此模式，直到卸载为止。无论设备后续是否升级为搭载 Android 10 或更高版本，或者应用后续是否更新为以 Android 10 或更高版本为目标平台，此兼容性行为均适用。

意思就是在新系统新安装的应用才会启用，覆盖安装会保持传统分区模式，例如：

* 系统通过 OTA 升级到 Android 10/11

* 应用通过更新升级到 targetSdkVersion >= 29 



### 补充

Q：之前讨论过一些问题，APP 无需权限可以访问自己创建的媒体，那么系统如何进行判断？

A：创建媒体时系统会给媒体打上 packageName tag，应用被卸载则会清除 tag ，所以不会存在使用同样 packageName 进行欺骗的情况。

Q：我可以在媒体集文件夹下创建文档，就可以避开权限的问题了？

A：官方文档上写了只能创建相应类型的媒体/文件，具体如何限制的，没有说明。



## 总结

从 Android 10提出分区存储之后到现在已经一年多了，所以Google 从强制推行的态度到现在  targetSDK >=30 才强制启用分区存储来看，Google 还是渐渐地选择给开发者留更多的时间。缺点当然是不强制启用的话，国内 APP 适配进度估计得延后了。不过好消息是在查资料的时候，看到了国内大厂的相关适配文章，至少说明大厂在跟进了。

去年（19年）的文档描述是无论 targetSDK 多少，明年（20年）高版本强制启用。

![scoped-storage-3](/Users/jinpengxie/Documents/Android/XLearnNotes/ScopedStorage/scoped-storage-3.png)

今年（20）文档描述是 targetSDK >=30 才强制启用

![scoped-storage-4](/Users/jinpengxie/Documents/Android/XLearnNotes/ScopedStorage/scoped-storage-4.png)



##### 关于适配的难度：

对绝对路径相关接口依赖比较深的 APP 适配还是改动挺多的；其次权限的划分很细，什么时候需要什么权限以及调用哪个接口，理解起来需要一定时间；`MediaStore API`  `SAF API` 这类接口以前就设计好了，我也觉得也不算特别友好；最后测试也需要重新进行。

所以虽然明年才会强制执行分区存储，但还是建议尽早理解和 review 项目中需要适配的代码。



##### 参考文档：

* [https://developer.android.com/preview/privacy/storage (Android 11 中的存储机制更新)](https://developer.android.com/preview/privacy/storage)  

* [https://developer.android.com/training/data-storage/shared/documents-files (从共享存储空间访问文档和其他文件)](https://developer.android.com/training/data-storage/shared/documents-files)  
* [https://developer.android.com/training/data-storage/files/external-scoped?hl=zh-cn (管理分区外部存储访问)](https://developer.android.com/training/data-storage/files/external-scoped?hl=zh-cn)  
* [https://developer.android.com/guide/topics/providers/document-provider (使用存储访问框架打开文件)](https://developer.android.com/guide/topics/providers/document-provider)
* [https://juejin.im/post/6844904063432130568 (Android 10 分区存储介绍及百度APP适配实践)](https://juejin.im/post/6844904063432130568)
* [https://github.com/android/storage-samples](https://github.com/android/storage-samples)