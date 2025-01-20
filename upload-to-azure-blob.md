# Azure upload to blob storage
### How to implement an image upload behaviour in Serenity.

## Implement IUploadStorage method

```
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Models;
using Azure.Storage.Blobs.Specialized;
using Microsoft.Extensions.Options;
using Serenity.Web;
using System;
using System.Collections.Generic;
using System.IO;
using System.Threading.Tasks;

namespace MovieTutorialNet6.Web.Modules.Common.AzureStorage
{
    public class UploadAzureStorage : IUploadStorage
    {
        private readonly IOptions<AzureStorageOptions> options;
        private BlobContainerClient blobContainer { get; }

        public UploadAzureStorage(IOptions<AzureStorageOptions> options)
        {
            this.options = options;

            blobContainer = new BlobContainerClient(
                options.Value.ConnectString,
                options.Value.ContainerName);
            
        }

        public string ArchiveFile(string path)
        {
            throw new System.NotImplementedException();
        }

        public string CopyFrom(IUploadStorage sourceStorage, string sourcePath, string targetPath, bool? autoRename)
        {
            AzureCopyFrom(sourcePath, targetPath);

            AzureCopyFrom(
                UploadPathHelper.GetThumbnailName(sourcePath),
                UploadPathHelper.GetThumbnailName(targetPath));

            return targetPath;
        }

        private string AzureCopyFrom(string sourcePath, string targetPath)
        {
            var sourceBlob = blobContainer.GetBlobClient(sourcePath);
            var lease = sourceBlob.GetBlobLeaseClient();
            lease.Acquire(TimeSpan.FromSeconds(-1));
            var destBlob = blobContainer.GetBlobClient(targetPath);
            var status = destBlob.StartCopyFromUri(sourceBlob.Uri);
            status.WaitForCompletion();
            while (status.HasCompleted == false)
            {
                Task.Run(async () =>
                {
                    await Task.Delay(200);
                });
            }
            lease.Break();
            return targetPath;
        }

        public void DeleteFile(string path)
        {
            AzureDelete(path);
            AzureDelete(UploadPathHelper.GetThumbnailName(path));
        }

        private void AzureDelete(string path)
        {
            var file = blobContainer.GetBlobClient(path);
            if (file.Exists()) file.Delete();
        }

        public bool FileExists(string path)
        {
            var exists = blobContainer.GetBlobClient(path).Exists();
            return exists;
        }

        public IDictionary<string, string> GetFileMetadata(string path)
        {
            var blockBlobClient = blobContainer.GetBlobClient(path);
            return blockBlobClient.GetProperties().Value.Metadata;

        }

        public string[] GetFiles(string path, string searchPattern)
        {
            throw new System.NotImplementedException();
        }

        public long GetFileSize(string path)
        {
            var blockBlobClient = blobContainer.GetBlobClient(path);
            return blockBlobClient.GetProperties().Value.ContentLength;
        }

        public string GetFileUrl(string path)
        {
            throw new System.NotImplementedException();
        }

        public Stream OpenFile(string path)
        {
            var blockBlobClient = blobContainer.GetBlobClient(path);
            return blockBlobClient.OpenRead();
        }

        public void PurgeTemporaryFiles()
        {
          
        }

        public void SetFileMetadata(string path, IDictionary<string, string> metadata, bool overwriteAll)
        {
            var blockBlobClient = blobContainer.GetBlobClient(path);
            blockBlobClient.SetMetadata(metadata);
            
        }

        public string WriteFile(string path, Stream source, bool? autoRename)
        {
            if (string.IsNullOrEmpty(path))
                throw new ArgumentNullException(nameof(path));

            try
            {
                var blockBlobClient = blobContainer.GetBlobClient(path);

                
                var writeStream = blockBlobClient.OpenWrite(true);
                source.CopyTo(writeStream);
                writeStream.Close();

                //this allow azure to remind a browser it's ok to show as a picture
                blockBlobClient.SetHttpHeaders(new BlobHttpHeaders { ContentType = "image/jpg" });
            }
            catch (Exception e)
            {
                Console.WriteLine(e.Message);
                return null;
            }

            return path;
        }
    }
}



```
## Link in `Startup.cs`
```
...
    services.AddSingleton<IUploadStorage UploadAzureStorage>();
...
```

## Usage is as normal

```
[ImageUploadEditor(FilenameFormat = "person/~")]
        [DisplayName("Image"), Size(200)]
        public string Image
        {
            get => fields.Image[this];
            set => fields.Image[this] = value;
        }