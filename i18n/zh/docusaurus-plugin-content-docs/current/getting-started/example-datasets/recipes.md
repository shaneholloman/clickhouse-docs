---
'description': 'RecipeNLG 数据集，包含 220 万条食谱'
'sidebar_label': '食谱数据集'
'slug': '/getting-started/example-datasets/recipes'
'title': '食谱数据集'
---

The RecipeNLG 数据集可以从 [这里](https://recipenlg.cs.put.poznan.pl/dataset) 下载。它包含 220 万个食谱，大小略小于 1 GB。

## 下载并解压数据集 {#download-and-unpack-the-dataset}

1. 访问下载页面 [https://recipenlg.cs.put.poznan.pl/dataset](https://recipenlg.cs.put.poznan.pl/dataset)。
2. 接受条款和条件并下载 zip 文件。
3. 可选：使用 `md5sum dataset.zip` 验证 zip 文件，应该等于 `3a168dfd0912bb034225619b3586ce76`。
4. 使用 `unzip dataset.zip` 解压 zip 文件。您将在 `dataset` 目录中得到 `full_dataset.csv` 文件。

## 创建表 {#create-a-table}

运行 clickhouse-client 并执行以下 CREATE 查询：

```sql
CREATE TABLE recipes
(
    title String,
    ingredients Array(String),
    directions Array(String),
    link String,
    source LowCardinality(String),
    NER Array(String)
) ENGINE = MergeTree ORDER BY title;
```

## 插入数据 {#insert-the-data}

运行以下命令：

```bash
clickhouse-client --query "
    INSERT INTO recipes
    SELECT
        title,
        JSONExtract(ingredients, 'Array(String)'),
        JSONExtract(directions, 'Array(String)'),
        link,
        source,
        JSONExtract(NER, 'Array(String)')
    FROM input('num UInt32, title String, ingredients String, directions String, link String, source LowCardinality(String), NER String')
    FORMAT CSVWithNames
" --input_format_with_names_use_header 0 --format_csv_allow_single_quote 0 --input_format_allow_errors_num 10 < full_dataset.csv
```

这是一个展示如何解析自定义 CSV 的示例，因为它需要多种调整。

说明：
- 数据集为 CSV 格式，但在插入时需要一些预处理；我们使用表函数 [input](../../sql-reference/table-functions/input.md) 来执行预处理；
- CSV 文件的结构在表函数 `input` 的参数中指定；
- 字段 `num`（行号）是多余的 - 我们从文件解析并忽略它；
- 我们使用 `FORMAT CSVWithNames` 但 CSV 中的表头将被忽略（通过命令行参数 `--input_format_with_names_use_header 0`），因为表头不包含第一个字段的名称；
- 文件仅使用双引号来括起来 CSV 字符串；一些字符串没有用双引号括起来，单引号不能被解析为字符串的括起来 - 这就是为什么我们还添加 `--format_csv_allow_single_quote 0` 参数；
- CSV 中的一些字符串无法解析，因为它们的值开头包含 `\M/` 序列； CSV 中唯一以反斜杠开头的值可以是 `\N`，被解析为 SQL NULL。我们添加 `--input_format_allow_errors_num 10` 参数，可以跳过最多十条格式错误的记录；
- 成分、方向和 NER 字段都有数组；这些数组以不寻常的形式表示：它们被序列化为字符串作为 JSON，然后放在 CSV 中 - 我们将它们解析为 String，然后使用 [JSONExtract](../../sql-reference/functions/json-functions.md) 函数将其转换为 Array。

## 验证插入的数据 {#validate-the-inserted-data}

通过检查行数：

查询：

```sql
SELECT count() FROM recipes;
```

结果：

```text
┌─count()─┐
│ 2231142 │
└─────────┘
```

## 示例查询 {#example-queries}

### 按食谱数量排序的顶级成分： {#top-components-by-the-number-of-recipes}

在这个示例中，我们学习如何使用 [arrayJoin](../../sql-reference/functions/array-join.md) 函数将数组扩展为一组行。

查询：

```sql
SELECT
    arrayJoin(NER) AS k,
    count() AS c
FROM recipes
GROUP BY k
ORDER BY c DESC
LIMIT 50
```

结果：

```text
┌─k────────────────────┬──────c─┐
│ salt                 │ 890741 │
│ sugar                │ 620027 │
│ butter               │ 493823 │
│ flour                │ 466110 │
│ eggs                 │ 401276 │
│ onion                │ 372469 │
│ garlic               │ 358364 │
│ milk                 │ 346769 │
│ water                │ 326092 │
│ vanilla              │ 270381 │
│ olive oil            │ 197877 │
│ pepper               │ 179305 │
│ brown sugar          │ 174447 │
│ tomatoes             │ 163933 │
│ egg                  │ 160507 │
│ baking powder        │ 148277 │
│ lemon juice          │ 146414 │
│ Salt                 │ 122558 │
│ cinnamon             │ 117927 │
│ sour cream           │ 116682 │
│ cream cheese         │ 114423 │
│ margarine            │ 112742 │
│ celery               │ 112676 │
│ baking soda          │ 110690 │
│ parsley              │ 102151 │
│ chicken              │ 101505 │
│ onions               │  98903 │
│ vegetable oil        │  91395 │
│ oil                  │  85600 │
│ mayonnaise           │  84822 │
│ pecans               │  79741 │
│ nuts                 │  78471 │
│ potatoes             │  75820 │
│ carrots              │  75458 │
│ pineapple            │  74345 │
│ soy sauce            │  70355 │
│ black pepper         │  69064 │
│ thyme                │  68429 │
│ mustard              │  65948 │
│ chicken broth        │  65112 │
│ bacon                │  64956 │
│ honey                │  64626 │
│ oregano              │  64077 │
│ ground beef          │  64068 │
│ unsalted butter      │  63848 │
│ mushrooms            │  61465 │
│ Worcestershire sauce │  59328 │
│ cornstarch           │  58476 │
│ green pepper         │  58388 │
│ Cheddar cheese       │  58354 │
└──────────────────────┴────────┘

50 rows in set. Elapsed: 0.112 sec. Processed 2.23 million rows, 361.57 MB (19.99 million rows/s., 3.24 GB/s.)
```

### 最复杂的草莓食谱 {#the-most-complex-recipes-with-strawberry}

```sql
SELECT
    title,
    length(NER),
    length(directions)
FROM recipes
WHERE has(NER, 'strawberry')
ORDER BY length(directions) DESC
LIMIT 10
```

结果：

```text
┌─title────────────────────────────────────────────────────────────┬─length(NER)─┬─length(directions)─┐
│ Chocolate-Strawberry-Orange Wedding Cake                         │          24 │                126 │
│ Strawberry Cream Cheese Crumble Tart                             │          19 │                 47 │
│ Charlotte-Style Ice Cream                                        │          11 │                 45 │
│ Sinfully Good a Million Layers Chocolate Layer Cake, With Strawb │          31 │                 45 │
│ Sweetened Berries With Elderflower Sherbet                       │          24 │                 44 │
│ Chocolate-Strawberry Mousse Cake                                 │          15 │                 42 │
│ Rhubarb Charlotte with Strawberries and Rum                      │          20 │                 42 │
│ Chef Joey's Strawberry Vanilla Tart                              │           7 │                 37 │
│ Old-Fashioned Ice Cream Sundae Cake                              │          17 │                 37 │
│ Watermelon Cake                                                  │          16 │                 36 │
└──────────────────────────────────────────────────────────────────┴─────────────┴────────────────────┘

10 rows in set. Elapsed: 0.215 sec. Processed 2.23 million rows, 1.48 GB (10.35 million rows/s., 6.86 GB/s.)
```

在这个示例中，我们使用 [has](../../sql-reference/functions/array-functions.md#hasarr-elem) 函数按数组元素进行过滤，并根据方向的数量进行排序。

有一个婚礼蛋糕需要完整的 126 个步骤来制作！展示这些步骤：

查询：

```sql
SELECT arrayJoin(directions)
FROM recipes
WHERE title = 'Chocolate-Strawberry-Orange Wedding Cake'
```

结果：

```text
┌─arrayJoin(directions)───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Position 1 rack in center and 1 rack in bottom third of oven and preheat to 350F.                                                                                           │
│ Butter one 5-inch-diameter cake pan with 2-inch-high sides, one 8-inch-diameter cake pan with 2-inch-high sides and one 12-inch-diameter cake pan with 2-inch-high sides.   │
│ Dust pans with flour; line bottoms with parchment.                                                                                                                          │
│ Combine 1/3 cup orange juice and 2 ounces unsweetened chocolate in heavy small saucepan.                                                                                    │
│ Stir mixture over medium-low heat until chocolate melts.                                                                                                                    │
│ Remove from heat.                                                                                                                                                           │
│ Gradually mix in 1 2/3 cups orange juice.                                                                                                                                   │
│ Sift 3 cups flour, 2/3 cup cocoa, 2 teaspoons baking soda, 1 teaspoon salt and 1/2 teaspoon baking powder into medium bowl.                                                 │
│ using electric mixer, beat 1 cup (2 sticks) butter and 3 cups sugar in large bowl until blended (mixture will look grainy).                                                 │
│ Add 4 eggs, 1 at a time, beating to blend after each.                                                                                                                       │
│ Beat in 1 tablespoon orange peel and 1 tablespoon vanilla extract.                                                                                                          │
│ Add dry ingredients alternately with orange juice mixture in 3 additions each, beating well after each addition.                                                            │
│ Mix in 1 cup chocolate chips.                                                                                                                                               │
│ Transfer 1 cup plus 2 tablespoons batter to prepared 5-inch pan, 3 cups batter to prepared 8-inch pan and remaining batter (about 6 cups) to 12-inch pan.                   │
│ Place 5-inch and 8-inch pans on center rack of oven.                                                                                                                        │
│ Place 12-inch pan on lower rack of oven.                                                                                                                                    │
│ Bake cakes until tester inserted into center comes out clean, about 35 minutes.                                                                                             │
│ Transfer cakes in pans to racks and cool completely.                                                                                                                        │
│ Mark 4-inch diameter circle on one 6-inch-diameter cardboard cake round.                                                                                                    │
│ Cut out marked circle.                                                                                                                                                      │
│ Mark 7-inch-diameter circle on one 8-inch-diameter cardboard cake round.                                                                                                    │
│ Cut out marked circle.                                                                                                                                                      │
│ Mark 11-inch-diameter circle on one 12-inch-diameter cardboard cake round.                                                                                                  │
│ Cut out marked circle.                                                                                                                                                      │
│ Cut around sides of 5-inch-cake to loosen.                                                                                                                                  │
│ Place 4-inch cardboard over pan.                                                                                                                                            │
│ Hold cardboard and pan together; turn cake out onto cardboard.                                                                                                              │
│ Peel off parchment.Wrap cakes on its cardboard in foil.                                                                                                                     │
│ Repeat turning out, peeling off parchment and wrapping cakes in foil, using 7-inch cardboard for 8-inch cake and 11-inch cardboard for 12-inch cake.                        │
│ Using remaining ingredients, make 1 more batch of cake batter and bake 3 more cake layers as described above.                                                               │
│ Cool cakes in pans.                                                                                                                                                         │
│ Cover cakes in pans tightly with foil.                                                                                                                                      │
│ (Can be prepared ahead.                                                                                                                                                     │
│ Let stand at room temperature up to 1 day or double-wrap all cake layers and freeze up to 1 week.                                                                           │
│ Bring cake layers to room temperature before using.)                                                                                                                        │
│ Place first 12-inch cake on its cardboard on work surface.                                                                                                                  │
│ Spread 2 3/4 cups ganache over top of cake and all the way to edge.                                                                                                         │
│ Spread 2/3 cup jam over ganache, leaving 1/2-inch chocolate border at edge.                                                                                                 │
│ Drop 1 3/4 cups white chocolate frosting by spoonfuls over jam.                                                                                                             │
│ Gently spread frosting over jam, leaving 1/2-inch chocolate border at edge.                                                                                                 │
│ Rub some cocoa powder over second 12-inch cardboard.                                                                                                                        │
│ Cut around sides of second 12-inch cake to loosen.                                                                                                                          │
│ Place cardboard, cocoa side down, over pan.                                                                                                                                 │
│ Turn cake out onto cardboard.                                                                                                                                               │
│ Peel off parchment.                                                                                                                                                         │
│ Carefully slide cake off cardboard and onto filling on first 12-inch cake.                                                                                                  │
│ Refrigerate.                                                                                                                                                                │
│ Place first 8-inch cake on its cardboard on work surface.                                                                                                                   │
│ Spread 1 cup ganache over top all the way to edge.                                                                                                                          │
│ Spread 1/4 cup jam over, leaving 1/2-inch chocolate border at edge.                                                                                                         │
│ Drop 1 cup white chocolate frosting by spoonfuls over jam.                                                                                                                  │
│ Gently spread frosting over jam, leaving 1/2-inch chocolate border at edge.                                                                                                 │
│ Rub some cocoa over second 8-inch cardboard.                                                                                                                                │
│ Cut around sides of second 8-inch cake to loosen.                                                                                                                           │
│ Place cardboard, cocoa side down, over pan.                                                                                                                                 │
│ Turn cake out onto cardboard.                                                                                                                                               │
│ Peel off parchment.                                                                                                                                                         │
│ Slide cake off cardboard and onto filling on first 8-inch cake.                                                                                                             │
│ Refrigerate.                                                                                                                                                                │
│ Place first 5-inch cake on its cardboard on work surface.                                                                                                                   │
│ Spread 1/2 cup ganache over top of cake and all the way to edge.                                                                                                            │
│ Spread 2 tablespoons jam over, leaving 1/2-inch chocolate border at edge.                                                                                                   │
│ Drop 1/3 cup white chocolate frosting by spoonfuls over jam.                                                                                                                │
│ Gently spread frosting over jam, leaving 1/2-inch chocolate border at edge.                                                                                                 │
│ Rub cocoa over second 6-inch cardboard.                                                                                                                                     │
│ Cut around sides of second 5-inch cake to loosen.                                                                                                                           │
│ Place cardboard, cocoa side down, over pan.                                                                                                                                 │
│ Turn cake out onto cardboard.                                                                                                                                               │
│ Peel off parchment.                                                                                                                                                         │
│ Slide cake off cardboard and onto filling on first 5-inch cake.                                                                                                             │
│ Chill all cakes 1 hour to set filling.                                                                                                                                      │
│ Place 12-inch tiered cake on its cardboard on revolving cake stand.                                                                                                         │
│ Spread 2 2/3 cups frosting over top and sides of cake as a first coat.                                                                                                      │
│ Refrigerate cake.                                                                                                                                                           │
│ Place 8-inch tiered cake on its cardboard on cake stand.                                                                                                                    │
│ Spread 1 1/4 cups frosting over top and sides of cake as a first coat.                                                                                                      │
│ Refrigerate cake.                                                                                                                                                           │
│ Place 5-inch tiered cake on its cardboard on cake stand.                                                                                                                    │
│ Spread 3/4 cup frosting over top and sides of cake as a first coat.                                                                                                         │
│ Refrigerate all cakes until first coats of frosting set, about 1 hour.                                                                                                      │
│ (Cakes can be made to this point up to 1 day ahead; cover and keep refrigerate.)                                                                                            │
│ Prepare second batch of frosting, using remaining frosting ingredients and following directions for first batch.                                                            │
│ Spoon 2 cups frosting into pastry bag fitted with small star tip.                                                                                                           │
│ Place 12-inch cake on its cardboard on large flat platter.                                                                                                                  │
│ Place platter on cake stand.                                                                                                                                                │
│ Using icing spatula, spread 2 1/2 cups frosting over top and sides of cake; smooth top.                                                                                     │
│ Using filled pastry bag, pipe decorative border around top edge of cake.                                                                                                    │
│ Refrigerate cake on platter.                                                                                                                                                │
│ Place 8-inch cake on its cardboard on cake stand.                                                                                                                           │
│ Using icing spatula, spread 1 1/2 cups frosting over top and sides of cake; smooth top.                                                                                     │
│ Using pastry bag, pipe decorative border around top edge of cake.                                                                                                           │
│ Refrigerate cake on its cardboard.                                                                                                                                          │
│ Place 5-inch cake on its cardboard on cake stand.                                                                                                                           │
│ Using icing spatula, spread 3/4 cup frosting over top and sides of cake; smooth top.                                                                                        │
│ Using pastry bag, pipe decorative border around top edge of cake, spooning more frosting into bag if necessary.                                                             │
│ Refrigerate cake on its cardboard.                                                                                                                                          │
│ Keep all cakes refrigerated until frosting sets, about 2 hours.                                                                                                             │
│ (Can be prepared 2 days ahead.                                                                                                                                              │
│ Cover loosely; keep refrigerated.)                                                                                                                                          │
│ Place 12-inch cake on platter on work surface.                                                                                                                              │
│ Press 1 wooden dowel straight down into and completely through center of cake.                                                                                              │
│ Mark dowel 1/4 inch above top of frosting.                                                                                                                                  │
│ Remove dowel and cut with serrated knife at marked point.                                                                                                                   │
│ Cut 4 more dowels to same length.                                                                                                                                           │
│ Press 1 cut dowel back into center of cake.                                                                                                                                 │
│ Press remaining 4 cut dowels into cake, positioning 3 1/2 inches inward from cake edges and spacing evenly.                                                                 │
│ Place 8-inch cake on its cardboard on work surface.                                                                                                                         │
│ Press 1 dowel straight down into and completely through center of cake.                                                                                                     │
│ Mark dowel 1/4 inch above top of frosting.                                                                                                                                  │
│ Remove dowel and cut with serrated knife at marked point.                                                                                                                   │
│ Cut 3 more dowels to same length.                                                                                                                                           │
│ Press 1 cut dowel back into center of cake.                                                                                                                                 │
│ Press remaining 3 cut dowels into cake, positioning 2 1/2 inches inward from edges and spacing evenly.                                                                      │
│ Using large metal spatula as aid, place 8-inch cake on its cardboard atop dowels in 12-inch cake, centering carefully.                                                      │
│ Gently place 5-inch cake on its cardboard atop dowels in 8-inch cake, centering carefully.                                                                                  │
│ Using citrus stripper, cut long strips of orange peel from oranges.                                                                                                         │
│ Cut strips into long segments.                                                                                                                                              │
│ To make orange peel coils, wrap peel segment around handle of wooden spoon; gently slide peel off handle so that peel keeps coiled shape.                                   │
│ Garnish cake with orange peel coils, ivy or mint sprigs, and some berries.                                                                                                  │
│ (Assembled cake can be made up to 8 hours ahead.                                                                                                                            │
│ Let stand at cool room temperature.)                                                                                                                                        │
│ Remove top and middle cake tiers.                                                                                                                                           │
│ Remove dowels from cakes.                                                                                                                                                   │
│ Cut top and middle cakes into slices.                                                                                                                                       │
│ To cut 12-inch cake: Starting 3 inches inward from edge and inserting knife straight down, cut through from top to bottom to make 6-inch-diameter circle in center of cake. │
│ Cut outer portion of cake into slices; cut inner portion into slices and serve with strawberries.                                                                           │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

126 rows in set. Elapsed: 0.011 sec. Processed 8.19 thousand rows, 5.34 MB (737.75 thousand rows/s., 480.59 MB/s.)
```
