# Use appsetting in typescript client
This is a bit hacky but seems to work ok. It's based on getting the appsettings into a global html space to be referenceable anywhere in the project.

## _layoutHead.cshtml
```
@using Microsoft.Extensions.Configuration
@inject IConfiguration Configuration
...
...

<script type="text/javascript">
   ...
    window["UploadBaseUrl"] = "@Configuration["UploadSettings:Url"]";
  
</script>
```

## use directly in .ts
```
let src = window["UploadBaseUrl"] + file;
```
