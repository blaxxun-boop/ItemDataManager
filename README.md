# Item Data Manager

Can be used to store data on items for your mod in Valheim.

## How to add item data

Custom item data can be attached to any item as well as item prefabs.

Custom item data is preserved while instantiating prefabs and upgrading items. Items with custom item data are also not stackable by default to prevent loss of item data.

### Merging the DLL into your mod

Download the ItemDataManager.dll from the release section to the right.
Including the DLL is best done via ILRepack (https://github.com/ravibpatel/ILRepack.Lib.MSBuild.Task). You can load this package (ILRepack.Lib.MSBuild.Task) from NuGet.

If you have installed ILRepack via NuGet, simply create a file named `ILRepack.targets` in your project and copy the following content into the file.

```xml
<?xml version="1.0" encoding="utf-8"?>
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
    <Target Name="ILRepacker" AfterTargets="Build">
        <ItemGroup>
            <InputAssemblies Include="$(TargetPath)" />
            <InputAssemblies Include="$(OutputPath)ItemDataManager.dll" />
        </ItemGroup>
        <ILRepack Parallel="true" DebugInfo="true" Internalize="true" InputAssemblies="@(InputAssemblies)" OutputFile="$(TargetPath)" TargetKind="SameAsPrimaryAssembly" LibraryPath="$(OutputPath)" />
    </Target>
</Project>
```

Make sure to set the ItemDataManager.dll in your project to "Copy to output directory" in the properties of the DLL and to add a reference to it.
After that, simply add `using ItemDataManager;` to your mod and use the `Data()` extension method on `ItemDrop.ItemData`, to localize your mod.

### Attaching arbitrary string data

String data is accessible by `itemdata.Data()["MyKey"]`. A `Remove("MyKey")` method also exists to clear data.

```cs
using ItemDataManager;

ItemDrop.ItemData itemData = itemDataToChange;
itemData.Data()["bonus"] = "1"; // saves "bonus" = "1" for your mod.
string currentValue = itemData.Data()["bonus"]; // reading a key "bonus"
itemData.Data().Remove("bonus"); // and also remove it
```

Note that custom data is scoped to your current mod GUID. It is also possible to access another mods data by explicitly requesting its data:

```cs
itemData.Data("my.other.mod.guid")["bonus"]
```

Usage of string data is recommended in case no extra handling needs to happen when an item is loaded or modified.

### Attaching a custom class to an item

Any custom class attachable to an item must inherit from `ItemDataManager.ItemData`.

These classes can be added and accessed via some methods on the `Data()` object.
```cs
MyData? mydata = itemData.Data().Add<MyData>(); // returns null if already exists
mydata = itemData.Data().GetOrCreate<MyData>(); // never fails
mydata = itemData.Data().Get<MyData>(); // returns null if missing
bool success = itemData.Data().Remove<MyData>();
```

That class has a string `Value` property, which directly reads and writes from the m_customData on the item.

That class exposes a few virtual methods which can be overridden:

```cs
// Checked if TryStack is not overridden
protected override bool AllowStackingIdenticalValues { get; set; };
// Called when the ItemData object is Add<>()'ed
public override void FirstLoad();
// Called on first access
public override void Load();
// Called right before the item is serialized
public override void Save();
// Called when the ItemData object is Remove()'d
public override void Unload();
// Called when an user upgrades the item
public override void Upgraded();
// Called when an item is tried to be stacked
public override string? TryStack(ItemData? data);
```

As a simple example, let's implement a container for a list of items:
```cs
public class ItemContainer : ItemData
{
	public List<string> items = new();

	public override void Save()
	{
		Value = string.Join(",", items);
	}

	public override void Load()
	{
		socketedGems = Value == "" ? new List<string>() : Value.Split(',').ToList();
	}
}
```

Now, if we have an item prefab for that, we can just attach this class to it directly:
```cs
prefab.GetComponent<ItemDrop>().m_itemData.Data().Add<ItemContainer>();
```

And all instances of this item will possess this ItemData.

The `ItemDrop.ItemData` instance itself is available via the `Item` property on this class.

#### Saving data

Generally it is unnecessary to manually call `Save()` on `Data()` or the `ItemData`. It will always be called when an item in an Inventory (player inventory, chests, ...) is saved.

However, prefabs will not be saved automatically after the initial ItemData add, nor will changes to items on the ground be saved automatically.
To save changes to items on ground, call `Save()` on the ItemDrop directly. To save changes to prefabs, call `Data().Save()`.

#### Stacking items

Items with custom data are generally not stacked, except if the `TryStack()` method is overridden or otherwise, `protected override bool AllowStackingIdenticalValues { get; set; } = true;` is set on the class and the `Value` of the ItemData objects are equal.

`TryStack(ItemData? data)` gives you the item the current item is tried to be stacked against. Return null if both items are not stackable against each other. Otherwise, return the new string value of the item. (e.g. `Save(); return Value;`).

#### Applying changes to items

If your ItemData class is not just storing data on an item, but also changing the item itself, it is necessary to register the class with the ItemDataManager so that it will be loaded upon instantiation:
```cs
ItemDataManager.ItemInfo.ForceLoadTypes.Add(typeof(MyItemData));
```

When such an item is cloned, it might have applied its changes to the original, but it might be undesirable to apply the changes twice. To check within `Load()` whether the ItemData already had been loaded, check for the `IsCloned` property.
