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
