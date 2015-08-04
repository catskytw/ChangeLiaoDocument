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
- BiasFX 的Dropbox Backup/Restore 共計有三個部份: 
  - 選單的增加
  - Bank backup/restore
  - Preset backup/restore
- 選單的增加:  
  - Setting Table 裡加入Dropbox 選單。Setting Table 選擇是依據Apple  [Settings Application Schema](https://developer.apple.com/library/ios/documentation/PreferenceSettings/Conceptual/SettingsApplicationSchemaReference/Introduction/Introduction.html) 完成。 
  - Bank Table 的資料是儲存在`[NSUserDefaults standardUserDefaults]` 內，key值為 `SoundFlow.Preset.Tab.{sequential_number}.BankName`， value 是 `{BankName}` (e.g. Factory, User...)

- Preset backup/restore:  
  - Preset Backup: 所有Preset 均放在`~/Document/Preset/{bank_name}/Preset.{sequential_number}.iTonesPreset` 來存放。將`~/Document/Preset` 一併壓縮上傳到Dropbox, 命名為 `tmp_presetBackup.zip`
  - Preset Restore: 將tmp_presetBackup.zip 解壓之 `~/Document/Preset` 之下
壓縮及解壓sample code:
```objective-c
-(void)restoreFromDropbox:(void(^)(void))downloadCompletedBlock restoreCompleted:(void(^)(void))restoreCompleteBlock{
    downloading = YES;
    uploadedFile = NO;
    self.downloadCompletedBlock = downloadCompletedBlock;
    self.restoreCompletedBlock = restoreCompleteBlock;
    [SVProgressHUD showWithStatus:@"Restoring" maskType:SVProgressHUDMaskTypeGradient];
    
    if ([self isNetworkReachbility]) {
        [self.restClient loadMetadata:@"/"];
    }else{
        [SVProgressHUD showErrorWithStatus:@"Failed to restore Preset Bank."];
        downloading = NO;
    }
}

-(void)backupPresetFileToDropbox{
    uploading = YES;
    downloading = NO;
    [SVProgressHUD showWithStatus:@"Backuping" maskType:SVProgressHUDMaskTypeGradient];

    //backup Preset Bank list
    if ([[PresetManager defaultManager] backupPresetBankKeyValues] && [self isNetworkReachbility]) {
        [self.restClient loadMetadata:@"/"];
    }else{
        [SVProgressHUD showErrorWithStatus:@"Failed to backup Preset Bank."];
        uploading = NO;
    }
}

- (void)restClient:(DBRestClient*)client loadedMetadata:(DBMetadata*)metadata {
    fileHash = nil;
    fileHash = metadata.hash;
    NSArray* validExtensions = [NSArray arrayWithObjects:@"zip", nil];
    NSMutableArray* presetZipPaths = [NSMutableArray new];
    if (metadata.isDirectory) {
        for (DBMetadata* child in metadata.contents) {
            NSString* extension = [[child.path pathExtension] lowercaseString];
            if (!child.isDirectory && [validExtensions indexOfObject:extension] != NSNotFound) {
                [presetZipPaths addObject:child.path];
                uploadedFile = child;
            }else
                uploadedFile = nil;
        }
    }
    if (uploading) {
        NSString* uploadPath = [self zipPresetFiles];
        [self.restClient uploadFile:@(UPLOAD_PRESET_ZIP_PATH)
                             toPath:@"/"
                      withParentRev:uploadedFile?uploadedFile.rev:nil
                           fromPath:uploadPath];

    }else if (downloading){
        if (uploadedFile) {
            NSString *intoPath = [[NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES) lastObject] stringByAppendingPathComponent:@(DOWNLOAD_PRESET_ZIP_PATH)];
            [self.restClient loadFile:uploadedFile.path atRev:uploadedFile.rev intoPath:intoPath];
        }else{
            NSLog(@"no uploaded file");
            [SVProgressHUD showErrorWithStatus:@"No file to restore"];
        }
        
    }
    
    filePaths = nil;
    filePaths = presetZipPaths;

}

```

- Bank backup/restore
  - 由於需要在restore 時，一併將Bank 回復，因此上述的值取出另存於一 `NSMutableDictionary` 後，寫回一個檔案名為 `PresetBankList`，此檔案亦一併存入 `tmp_presetBackup.zip` 。
下圖是tmp_presetBackup.zip 解開之範例。

![](https://raw.githubusercontent.com/catskytw/ChangeLiaoDocument/master/tmpPresetBackup.png)

- 取出所有Bank Name 的method:
```objective-c
- (NSArray *)allPresetBankKeys{
    NSMutableArray *keys = [NSMutableArray array];
    int i = 0;
    while (1) {
        NSString *assumeKey = [NSString stringWithFormat:@"SoundFlow.Preset.Tab.%d.BankName", i];
        if ([AppPreference stringValueOf:assumeKey withDefault:nil]) {
            [keys addObject:assumeKey];
        }else
        break;
        i++;
    }
    return keys;
}
```

  - Backup/Restore Bank name 的sample code
```objective-c
- (BOOL)backupPresetBankKeyValues{
    NSArray *bankList = [self allPresetBankKeys];
    __block NSMutableDictionary *bankDic = [NSMutableDictionary dictionary];
    
    //fetching value from NSUserDefault
    [bankList enumerateObjectsUsingBlock:^(NSString *bankName, NSUInteger idx, BOOL *stop){
        [bankDic setValue:[AppPreference stringValueOf:bankName withDefault:nil]
                   forKey:bankName];
    }];
    
    //make sure [UIApplication presetPath] exists.
    BOOL isDir = YES;
    if (![[NSFileManager defaultManager] fileExistsAtPath:[UIApplication presetPath] isDirectory:&isDir]) {
        [[NSFileManager defaultManager] createDirectoryAtPath:[UIApplication presetPath] withIntermediateDirectories:YES attributes:nil error:nil];
    }
    //remove the file if it exists.
    NSString *filePath = [[UIApplication presetPath] stringByAppendingPathComponent:@"PresetBankList"];
    if ([[NSFileManager defaultManager] fileExistsAtPath:filePath]){
        [[NSFileManager defaultManager] removeItemAtPath:filePath error:nil];
        //Change: add DDLog here?
    }
    
    //write back to file under ~/Document/Preset, named 'PresetBankList'
    BOOL isWriteBack = [bankDic writeToFile:filePath atomically:YES];
    return isWriteBack;
}

- (BOOL)restorePresetBankKeyValues:(NSString *)filePath{
    BOOL r = NO;
    //remove all BankList
    NSArray *bankList = [self allPresetBankKeys];
    [bankList enumerateObjectsUsingBlock:^(NSString *bankName, NSUInteger idx, BOOL *stop){
        [[NSUserDefaults standardUserDefaults] removeObjectForKey:bankName];
    }];
    
    //write back NSUserDefault
    NSDictionary *dictFromFile = [[NSMutableDictionary alloc] initWithContentsOfFile:filePath];
    if (dictFromFile) {
        [[dictFromFile allKeys] enumerateObjectsUsingBlock:^(NSString *key, NSUInteger idx, BOOL *stop){
            [AppPreference setStringValue:[dictFromFile valueForKey:key] of:key];
        }];
        r = YES;
    }
    
    return r;
}

```

