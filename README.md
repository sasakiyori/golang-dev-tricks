# golang-dev-tricks

Tricks for golang coders in real development scenarios!

## Get file size when writing

```go
func GetFileSizeWhenWriting() {
    f, err := os.Create("/path/to/file")
    if err != nil {
        panic(err)
    }
    defer f.Close()

    _, err := f.WriteString("blabla")
    if err != nil {
        panic(err)
    }
    _, err := OuterWritingFunc(f)
    if err != nil {
        panic(err)
    }

    size, err := f.Seek(0, io.SeekCurrent)
    if err != nil {
        panic(err)
    }
    fmt.Println("get current file size:", size)
}
```

## Upload data stream directly to S3 instead of memory buffer cache

```go
func UploadDirectlyToS3() {
    sess, err := session.NewSession(&aws.Config{})
    if err != nil {
        panic(err)
    }
    cli := s3.New(sess)
    uploader := s3manager.NewUploaderWithClient(cli)
    pr, pw := io.Pipe()
    params := &s3manager.UploadInput{
        Bucket: aws.String("blabla"),
        Key: aws.String("blabla"),
        Body: pr,
    }

    ch := make(chan error)
    defer close(ch)
    // pipe reader
    go func() {
        // it will stuck until io pipe is broken(closed)
        _, err := uploader.UploadWithContext(ctx, params)
        if err != nil {
            pr.Close()
        }
        ch <- err
    }()
    // pipe writer
    func(){
        defer pw.Close()
        _, err := OuterWritingFunc(pw)
        if err != nil {
            fmt.Println(err.Error())
        }
    }()

    if uploadErr := <-ch; uploadErr != nil {
        fmt.Println(uploadErr.Error())
    }
}
```

## Watch file change by k8s configmap

```shell
# Event trigger list when files in directory changed by k8s configmap
CREATE        "app/..2023_12_21_11_49_43.4178923114" 
CHMOD         "app/..2023_12_21_11_49_43.4178923114"
CREATE        "app/..data_tmp"
RENAME        "app/..data_tmp"
CREATE        "app/..data"
REMOVE        "app/..2023_12_21_11_45_32.3293038872"
```

```go
func WatchFileChangeByK8sConfigMap() {
    watcher, err := fsnotify.NewWatcher()
    if err != nil {
        panic(err)
    }
    defer watcher.Close()
    if err = watcher.Add("/path/to/watch"); err != nil {
        panic(err)
    }
    for {
        select {
        case event, ok := <-watcher.Events:
            if !ok {
                continue
            }
            if event.Op.Has(fsnotify.Remove) {
                // file changed
            }
        case wErr, ok := <-watcher.Errors:
            if !ok {
                continue
            }
            if wErr != nil {
                fmt.Println(wErr.Error())
            }
        }
    }
}
```

## Scan fields by gorm but do not create columns in DB

```go
type Table struct {
    // ref: https://github.com/go-gorm/gorm/issues/5178
    // <-:false             no write permission
    // ->                   read permission
    // -:migration          ignore migration
    A int `json:"a" gorm:"<-:false;->;-:migration"`
}
```

## Streaming upload file in http client side

```go
func StreamingUpload() error {
    pr, pw := io.Pipe()
    writer := multipart.NewWriter(pw)
    go func() {
        // optional extra form data field
        writer.WriteField("xxx", "xxx")
        file, err := os.Open(filePath)
        if err != nil {
            pw.CloseWithError(err)
            return
        }
        defer file.Close()

        part, err := writer.CreateFromFile("file", fileName)
        if err != nil {
            pw.CloseWithError(err)
            return
        }
        _, err = io.Copy(part, file)
        if err != nil {
            pw.CloseWithError(err)
            return
        }
        pw.CloseWithError(writer.Close())
    }()

    req, err := http.NewRequest("POST", url, pr)
    if err != nil {
        return err
    }
    req.Header.Set("Content-Type", writer.FormDataContentType())
    client := &http.Client{Timeout: 0}
    resp, err := client.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    
    // ...
    return nil
}
```

## Dynamic Time Ticker

```go
func DynamicTimeTicker(ctx context.Context, trigger <-chan struct{}) {
    interval := 60
    ticker := time.NewTicker(time.Second * time.Duration(interval))
    loop:
    for {
        select {
        case <-ctx.Done():
            break loop
        case <-trigger:
            if interval == 1 {
                break loop
            }
            fmt.Println("TRIGGER")

            ticker.Stop()
            interval /= 2
            ticker = time.NewTicker(time.Second * time.Duration(interval))
        case <-ticker.C:
            fmt.Println("TICK")
        }
    }

    ticker.Stop()
}
```
