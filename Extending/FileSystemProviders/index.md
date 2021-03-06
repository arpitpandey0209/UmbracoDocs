---
versionFrom: 8.0.0
meta.Title: "Umbraco File System Providers"
meta.Description: "A guide to creating custom file systems in Umbraco"
---

# Custom file systems (IFileSystem)

## MediaFileSystem
By default, Umbraco uses an instance of `PhysicalFileSystem` to handle the storage location of the media archive (~/media).

This can be configured during composition:

```csharp
using Umbraco.Core.Composing;
using Umbraco.Core;
using Umbraco.Core.IO;

namespace Umbraco8Examples.Composition
{
    public class SetMediaFileSystemComposer : IUserComposer
    {
        public void Compose(Umbraco.Core.Composing.Composition composition)
        {
            composition.SetMediaFileSystem(() => new PhysicalFileSystem("~/custommediafolder"));
        }
    }
}
```
The `PhysicalFileSystem` accepts the application path to the 'virtual Root' location of the Media folder. The ~ is therefore important!

Alternatively you can combine these two:

**RootPath** - the rooted, filesystem path, using directory separator chars, NOT ending with a separator // eg "c:" or "c:\path\to\site" or "\\server\path"
and **RootUrl** - the relative url, using url separator chars, NOT ending with a separator // eg "" or "/Views" or "/Media" or "/<vpath>/Media" in case of a virtual path

```csharp
 composition.SetMediaFileSystem(() => new PhysicalFileSystem("Z:\Storage\UmbracoMedia","http://media.example.com/media" ));
```
:::note
In Umbraco V7, this configuration was located a physical file: `/config/FileSystemProviders.config`.
:::

### IFileSystem

`PhysicalFileSystem` implements the `IFileSystem` interface, and it is possible to replace it with a custom class - eg. if you want your media files stored on Azure or something similar.

If you configure Umbraco to use a custom file system provider for media, you most likely won't need to access the implementation directly. Umbraco uses a wrapper class called `MediaFileSystem`. You can get a reference to this wrapper class with the following code:

```csharp
IMediaFileSystem media = Current.MediaFileSystem;
```
or via dependency injection in the constructor for your custom class or controller:
```csharp
 public class ImagesController : UmbracoAuthorizedApiController
    {
        private readonly IMediaFileSystem _mediaFileSystem;


        public ImagesController(IMediaFileSystem mediaFileSystem)
        {
            _mediaFileSystem = mediaFileSystem;
        }
```

This will be enough in most cases and is the way Umbraco will access the file system provider. The wrapper class implements the same interface `IFileSystem` as any custom providers should do, so you will be able to call the same methods.

## MediaPath Scheme

The MediaPath Scheme defines the current set of rules that decide the format of the Media Path when it is saved into the media archive wherever it is located.

By default the MediaPath scheme used by Umbraco is the `UniqueMediaPathScheme` this generates a unique 'folder' to place the uploaded image in eg.

`/media/dozdrg2f/mylovelyimage.jpg`

`/media` is defined by the PhysicalFileSystem and 'dozdrg2f' is generated by the `UniqueMediaPathScheme`.

:::note
In Umbraco 7 the integer ids were used in the path, and this approach is still possible using the 'OriginalMediaPathScheme'
:::

You can set the `MediaPathScheme` during composition, for example if you wanted to revert back to the V7 methodology in a migrated site:
```
  composition.RegisterUnique<IMediaPathScheme, OriginalMediaPathScheme>();
```

And you could create your own logic for the path by implementing `IMediaPathScheme`

## Other IFileSystems

Umbraco also registers instances of PhysicalFileSystem for the following parts of Umbraco that persist to 'files':

- MacroPartialsFileSystem
- PartialViewsFileSystem
- StylesheetsFileSystem
- ScriptsFileSystem
- MvcViewsFileSystem 

These are accessible via properties on the current registered `IFileSystems` implementation.

```csharp
IFileSystem macroPartialsFileSystem = Current.FileSystems.MacroPartialsFileSystem;
```
or again via dependency injection

```csharp
  public class FileSystemLocations 
    {
        private readonly IFileSystems _fileSystems;
        public FileSystemLocations(IFileSystems fileSystems)
        {
            _fileSystems = fileSystems;
            var macroPartialsFileSystem = _fileSystems.MacroPartialsFileSystem;
        }        
```

Both `IFileSystem` and `IMediaFileSystem` are located in the `Umbraco.Core.IO` namespace.

## Custom providers

There is an Azure Blob Storage provider:

* [Azure Blob Storage](Azure-Blob-Storage/)

### Creating your own FileSystem provider
Create your filesystem class:
```csharp
            public class MyFileSystem : FileSystemWrapper
            {
                public MyFileSystem(IFileSystem innerFileSystem)
                    : base(innerFileSystem)
                { }
            }
```
The constructor can have more parameters, that will be resolved by the dependency injection container.

Register your new filesystem implementation, in a component:
```csharp
  composition.RegisterFileSystem<MyFileSystem>();
```
Register the underlying filesystem:
```csharp
  composition.RegisterUniqueFor<IFileSystem, MyFileSystem>(...);
```
You can inject `MyFileSystem` wherever it's needed.

Furthermore it's possible to declare a filesystem interface:
```csharp
  public interface IMyFileSystem : IFileSystem
  { }
```
Make the class implement the interface, then
register your filesystem, in a component:
```csharp
  composition.RegisterFileSystem<IMyFileSystem, MyFileSystem>();
  composition.RegisterUniqueFor<IFileSystem, IMyFileSystem>(...);
```
You can inject IMyFileSystem wherever it's needed.



