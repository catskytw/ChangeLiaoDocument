# Dropbox Framework Basic Operations
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

```objective-c
- (void)restClient:(DBRestClient *)client loadedFile:(NSString *)localPath
    contentType:(NSString *)contentType metadata:(DBMetadata *)metadata {
    NSLog(@"File loaded into path: %@", localPath);
}

- (void)restClient:(DBRestClient *)client loadFileFailedWithError:(NSError *)error {
    NSLog(@"There was an error loading the file: %@", error);
}
```

- 其他檔案操作(e.g. cancelUpload, thumbnail, createFolder, moveFrom..)請參閱 `DBRestClient.h`
  
  
  
# BiasFX 的Dropbox Backup/Restore
- Setting Table 裡加入Dropbox 選單。Setting Table 選擇是依據Apple  [Settings Application Schema](https://developer.apple.com/library/ios/documentation/PreferenceSettings/Conceptual/SettingsApplicationSchemaReference/Introduction/Introduction.html) 完成。 

- Preset 的Backup: 所有Preset 均放在`~/Document/Preset/{bank_name}/Preset.{sequential_number}.iTonesPreset` 來存放。將`~/Document/Preset` 一併壓縮上傳到Dropbox, 命名為 `tmp_presetBackup.zip`

- Bank Table:
  - Bank Table 的資料是儲存在`[NSUserDefaults standardUserDefaults]` 內，key值為 `SoundFlow.Preset.Tab.{sequential_number}.BankName`， value 是 `{BankName}` (e.g. Factory, User...)

  - 由於需要在restore 時，一併將Bank 回復，因此上述的值取出另存於一 `NSMutableDictionary` 後，寫回一個檔案名為 `PresetBankList`，此檔案亦一併存入 `tmp_presetBackup.zip` 。
下圖是tmp_presetBackup.zip 解開之範例。
