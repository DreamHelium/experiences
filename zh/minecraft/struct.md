# 结构文件

在Minecraft中，我们可能要用到各种各样的结构文件。常见的就是**原版NBT文件**（其中还有机械动力的蓝图文件）、**litematic投影文件**、**schema投影文件**。本文尽量以图文并貌的形式介绍这~三种~（成稿时暂时只研究了两种）文件，希望能对你有所帮助。

## 信息（原版NBT文件没有）

[metadata_pic](../../pic/litematic_metadata.png)

上图是**litematic投影文件**的一些Metadata信息，其实可以一目了解，除了包括了对应的版本以外，还有创建、修改的时间（以unix时间戳储存），还有大小、描述等信息，此处不再赘述。

另注：**原版NBT文件**中存在DataVersion的键，但是没有其他信息。

## 区域（litematic文件独有）

区域（**Regions**）在litematic文件的根下的Regions下，每个区域独有一个Compound结构。

## 区域信息

### 大小

在**litematic投影文件**中，大小以Size的Compound存在；而在**NBT文件**中，大小以size的*List*存在。其中还有差异的是，**litematic投影文件**有每个tag的键（x、y、z），而**NBT文件**中没有。

### 实体

TODO：此处用得比较少，暂不实现。

### 方块状态值（BlockStates）——litematic投影文件

此处以Long Array的值存在，以位的方式存储着每个方块的“偏移”（此处应为Palette, 但是不知道怎么翻译好）。~如果按照Java的存储方式（大端序），理应就是直接按位的方式直接存储。~（具体详见litematic项目源代码，此处仅为猜测）

以`dhlrc`项目的代码为例，如果想要取出对应方块的偏移值，则可以使用以下代码：

```C
/* The function below uses the implement from another project:
 * "litematica-tools" from KikuGie
 * https://github.com/Kikugie/litematica-tools
 * It uses MIT License, the license file could be found in config/
 * since files in config/ are also from this project.
 */

int lite_region_block_id(LiteRegion* lr, uint64_t index)
{
    int64_t* state = lr->states;
    int bits = lr->move_bits;
    uint64_t start_bit = index * bits;
    int start_state = start_bit / 64;
    int and_num = (1 << bits) - 1;
    int move_num = start_bit & 63;
    int end_num = start_bit % 64 + bits;
    int id = 0;
    if(end_num <= 64)
        id = (uint64_t)(state[start_state]) >> move_num & and_num;
    else
    {
        int move_num_2 = 64 - move_num;
        if( start_state + 1 >= lr->states_num)
            g_error("Out of range!");
        id = ((uint64_t)state[start_state] >> move_num | state[start_state + 1] << move_num_2)& and_num;
    }
    return id;
}
```

如果想要存回去，则需先把位对应的值塞到数组中，在`dhlrc`中，为了简化逻辑，是采用了`DhBit`结构（本质上还是数组，只是带了个偏移量），然后一位一位地塞进去：

```C
void dh_bit_push_back_bit(DhBit* bit, int b)
{
    if(bit->bits % 64 == 0)
    {
        bit->array = realloc(bit->array, (bit->bits / 64 + 1) * sizeof(int64_t));
        bit->array[bit->bits / 64] = 0;
    }
    bit->array[bit->bits / 64] |= ((int64_t)!!b << (bit->bits % 64));
    bit->bits++;
}
```

### PendingBlockTicks和PendingFluidTicks——litematic文件

未知。

### Position——litematic文件

与原点的位置偏移量。

### 方块“偏移”（Palette）

里面存储了方块的大部分信息（其实只有两部分，甚至只有一部分）：名称（**Name**）和属性（**Properties**）。

投影文件中的**BlockStatePalette**和NBT文件中的**palette**可以说是完全一致的实现，一般认为这种设计应该是源于存档的存储方式。

### 方块实体

NBT文件并未采用BlockStates的方式存储方块位置，相反，它采用了**blocks**的形式，内有**pos**（还是和之前size一样是无键的三个值组成的List），**state**（其实就是对应的palette），以及存放了实体信息的**nbt**。

litematic文件使用**TileEntities**来存储方块实体信息。与**nbt**不同的是，它只是多了表示位置的x、y、z键。

### 总览（上为NBT下为litematic）

[full_sight](../../pic/nbt_sight.png)

## 实现

在目前的`dhlrc`中，使用一个通用的`Region`结构来存储结构。它是将共通的部分进行存储，以减少~三~两种格式的差异，以便进行后续处理。

通用结构的总览（因为C语言没有泛形，所以里面的类型都是`GPtrArray`的typedef。~说实话要是有泛形我也可能要用typedef~）：

```C
typedef struct _Region
{
    /** The base information */
    int data_version;
    /** The size of region */
    RegionSize* region_size;
    /** The block info array */
    BlockInfoArray* block_info_array;
    /** The Palette info array*/
    PaletteArray* palette_array;
} Region;
```

`BlockInfo`和`Palette`的实现：

```C
typedef struct BlockInfo{
    int index;
    Pos* pos;
    char* id_name;
    int palette;
    NBT* nbt;
    NbtInstance* instance;
} BlockInfo;

typedef struct Palette{
    char* id_name;
    DhStrArray* property_name;
    DhStrArray* property_data;
} Palette;
```

具体如何兼容就不做具体介绍了，技术细节其实在上文均有阐述。

谢谢阅读！