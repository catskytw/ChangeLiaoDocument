- 下載Dropbox Core SDK (<https://www.dropbox.com/developers/reference/sdk>)

- 使用公司(或你自己的)dropbox 帳號, 在app console建立一個app
  - 記住key/secret, 最好是另外copy/paste 
  - 存取目錄permission type 有兩種
    - full dropbox: 所有的目錄與檔案都能看見
    - app folder: 僅能看見app的folder, 且無法與其他app共用
    
- 下載的SDK解開後, 將dropbox framework 拉到你的project 裡
  - Copy items into destination group's folder 勾選
  - project 加入security.framework
  
- 加入 `#import <DropboxSDK/DropboxSDK.h>`

- 啟始dropbox session, appKey/appSecret 填上剛申請app所屬的key/secret

- sample code:
```objective-c
    DBSession *session = [[DBSession alloc] initWithAppKey:appKey appSecret:appSecret root:root];
    session.delegate = self;
    [DBSession setSharedSession:session];
    [DBRequest setNetworkRequestDelegate:self];
    //delegate 的部份就依自己的情況指定, 另外callback method 就依自己需求實作
```

- 連結至dropbox, easy.
```objective-c
if(![[DBSession sharedSession] isLinked]){
       [[DBSession sharedSession] link];
   }
```

- 加入URL type.
  - Project Navigation -> TARGETS(選擇你要build 的target) -> Info -> URL type
  - 新增一筆
  - URL schemes 加入 db-yourAppKey(yourAppKey 請置換成申請時給予的app key)
  - 如下圖![](https://raw.githubusercontent.com/catskytw/ChangeLiaoDocument/master/dropboxURLType.jpg)
  
- 建立一個DBClient, 檔案以及目錄的操作都要依靠DBClient
```objective-c
    DBRestClient *dropboxClient =[[DBRestClient alloc] initWithSession:[DBSession sharedSession]];
    dropboxClient.delegate=self;

    //上傳
    NSString *localPath =[[NSBundle mainBundle] pathForResource:@"screenshot" ofType:@"png"];
    NSString *filename =@"screenshot.png";
    NSString *destDir =@"yourPath";
    
    [dropboxClient uploadFile:filename toPath:destDir withParentRev:nil fromPath:localPath];
    
    //下載
    [dropboxClient loadFile:@"yourFile" intoPath:@"detinationFile"];
```

- 其他檔案操作(e.g. cancelUpload, thumbnail, createFolder, moveFrom..)請參閱 `DBRestClient.h`
  
