![](https://zhuanlan.zhihu.com/p/532876904)  

本文梳理了Elasticsearch对于数值索引实现方案的升级和优化思考，从2015年至今数值索引的方案经历了多个版本的迭代，实现思路从最初的字符串模拟到KD-Tree，技术越来越复杂，能力越来越强大，应用场景也越来越丰富。从地理位置信息建模到多维坐标，数据检索到数据分析洞察都可以看到Elasticsearch的身影。

# 一、业务背景

LBS服务是当前互联网重要的一环，涉及餐饮、娱乐、打车、零售等场景。在这些场景中，有很重要的一项基础能力：搜索附近的POI。比如搜索附近的美食，搜索附近的电影院，搜索附近的专车，搜索附近的门店。例如：以某个坐标点为中心查询出1km半径范围的POI坐标，如下图所示：

![](https://pic2.zhimg.com/80/v2-5368e81b429171703ae24334309551e5_720w.webp)

Elasticsearch在地理位置信息检索上具备了毫秒级响应的能力，而毫秒级响应对于用户体验至关重要。上面的问题使用Elasticsearch，只需用到geo_distance查询就可以解决业务问题。使用Elasticsearch的查询语法如下：

```java
GET /my_locations/_search
{
  "query": {
    "bool": {
      "must": {
        "match_all": {}
      },
      "filter": {
        "geo_distance": {
          "distance": "1km",
          "pin.location": {
            "lat": 40,
            "lon": 116
          }
        }
      }
    }
  }
}
```

工具的使用是一个非常简单的事情，更有意思在于工具解决问题背后的思想。理解了处理问题的思想，就可以超然于工具本身，做到举一反三。本文基于在海量数据背景下，如何实现毫秒级搜索附近的POI这个问题，探讨了Elasticsearch的实现方案，以及实现地理位置索引技术的演进过程。

# 二、背景知识

在介绍Elasticsearch的处理方案前，我们首先需要介绍一些背景知识，主要是3个问题。

1.  **如何精确定位一个地址？**

由经度、纬度和相对高度组成的地理坐标系，能够明确标示出地球上的任何一个位置。地球上的经度范围[-180， 180]，纬度范围[-90，90]。通常以本初子午线(经度为0)、赤道(纬度为0)为分界线。对于大多数业务场景，由经纬度组成的二维坐标已经足以应对业务问题，可能重庆山城会有些例外。

2. **如何计算两个地址距离？**

对于平面坐标系，由勾股定理可以方便计算出两个点的距离。但是由于地球是一个不完美球体，且不同位置有不同海拔高度，所以精确计算两个距离位置是一个非常复杂的问题。在不考虑高度的情况下，二维坐标距离通常使用Haversine公式。

这个公式非常简单，只需用到arcsin和cos两个高中数学公式。其中φ和λ表示两个点纬度和经度的弧度制度量。其中d即为所求两个点的距离，对应的数学公式如下(参考维基百科)：

![](https://pic2.zhimg.com/80/v2-3346809df419043bc0ea4fec2c7cbe35_720w.webp)

  

程序员更喜欢看代码，对照代码理解公式更简单。相应的代码如下：

```java
// 代码摘自lucene-core-8.2.0， SloppyMath工具类
 
 /**
  * Returns the Haversine distance in meters between two points
  * given the previous result from {@link #haversinSortKey(double, double, double, double)}
  * @return distance in meters.
  */
 public static double haversinMeters(double sortKey) {
   return TO_METERS * 2 * asin(Math.min(1, Math.sqrt(sortKey * 0.5)));
 }
 
 /**
  * Returns a sort key for distance. This is less expensive to compute than
  * {@link #haversinMeters(double, double, double, double)}, but it always compares the same.
  * This can be converted into an actual distance with {@link #haversinMeters(double)}, which
  * effectively does the second half of the computation.
  */
 public static double haversinSortKey(double lat1, double lon1, double lat2, double lon2) {
   double x1 = lat1 * TO_RADIANS;
   double x2 = lat2 * TO_RADIANS;
   double h1 = 1 - cos(x1 - x2);
   double h2 = 1 - cos((lon1 - lon2) * TO_RADIANS);
   double h = h1 + cos(x1) * cos(x2) * h2;
   // clobber crazy precision so subsequent rounding does not create ties.
   return Double.longBitsToDouble(Double.doubleToRawLongBits(h) & 0xFFFFFFFFFFFFFFF8L);
 }
 // haversin
 // TODO: remove these for java 9, they fixed Math.toDegrees()/toRadians() to work just like this.
 public static final double TO_RADIANS = Math.PI / 180D;
 public static final double TO_DEGREES = 180D / Math.PI;
 
 // Earth's mean radius, in meters and kilometers; see http://earth-info.nga.mil/GandG/publications/tr8350.2/wgs84fin.pdf
 private static final double TO_METERS = 6_371_008.7714D; // equatorial radius
 private static final double TO_KILOMETERS = 6_371.0087714D; // equatorial radius
 
/**
  * Returns the Haversine distance in meters between two points
  * specified in decimal degrees (latitude/longitude).  This works correctly
  * even if the dateline is between the two points.
  * <p>
  * Error is at most 4E-1 (40cm) from the actual haversine distance, but is typically
  * much smaller for reasonable distances: around 1E-5 (0.01mm) for distances less than
  * 1000km.
  *
  * @param lat1 Latitude of the first point.
  * @param lon1 Longitude of the first point.
  * @param lat2 Latitude of the second point.
  * @param lon2 Longitude of the second point.
  * @return distance in meters.
  */
 public static double haversinMeters(double lat1, double lon1, double lat2, double lon2) {
   return haversinMeters(haversinSortKey(lat1, lon1, lat2, lon2));
 }
```

**3. 如何方便在互联网分享经纬度坐标？**

Geohash是2008-02-26由Gustavo Niemeyer在自己的个人博客上公布的算法服务。其初衷在于通过对经纬度的编码对外提供简短的URL标识地图位置，方便在电子邮件、论坛和网站中使用。

实际上Geohash的价值不仅仅是提供简短的URL，它更大的价值在于：

1.  Geohash给地图上每个坐标提供了独一无二的ID，这个唯一ID就相当于给每个地理位置提供了一个身份证。唯一ID在数据库中应用场景非常丰富。  
    
2.  在数据库中给坐标点提供了另一种存储方式，将二维的坐标点转化成为一维的字符串，对于一维数据就可以借助B树等索引来加速查询。  
    
3.  Geohash是一种前缀编码，位置相近的坐标点前缀相同。通过前缀提供了高性能的邻近位置POI查询，而邻近位置POI查询是LBS服务的核心能力。

关于Geohash的编码规则，这里不展开。这里最关键的点在于：

> Geohash是一种前缀编码，位置相近的坐标点前缀相同。Geohash编码长度不同，所覆盖区域范围不同。

在前面知识的铺垫下，最简单的求一个坐标点指定半径范围内的坐标集合的方案就出炉了。

-   **暴力算法**

> 中心坐标点依次跟集合中每个坐标点计算距离，筛选出符合半径条件的坐标点。

这个算法大家太熟悉了，就是最常见的暴力(Brute Force)算法。这个算法在海量数据背景下是没法满足毫秒级响应时间要求的，所以多用于离线计算。对于毫秒级响应的业务诉求，这个算法可以基于geohash进行改造。

-   **二次筛选**

1.  基于坐标中心点计算出geohash, 基于半径确定geohash前缀。
2.  通过Geohash前缀初筛出大致符合要求的坐标点(需要将中心点所在Geohash块周围8个Geohash块纳入初筛范围)。
3.  对于初筛结果使用Haversine公式进行二次筛选。

除了上述方案，Elasticsearch在地理信息处理上有哪些奇思妙想呢？

# 三、方案演进

Elasticsearch从2.0版本开始支持geo_distance查询，到当前已更新到7.14版本。

从2015年至今已经经历了6年的发展， 建设了如下的能力：

![](https://pic4.zhimg.com/80/v2-6b60e1a2bf063e2060b874603518cfaf_720w.webp)

技术迭代大致可以分为3个阶段：

![](https://pic2.zhimg.com/80/v2-2abfd259c06b626678d57fd2a5399bed_720w.webp)

发展的成效显著，从性能测试的结果可以略窥一二：

![](https://pic4.zhimg.com/80/v2-1a40d1cf12a2da9e771d1ea8841f750b_720w.webp)

![](https://pic3.zhimg.com/80/v2-f2ce022f1a4da75f19cc51b77727222a_720w.webp)

![](https://pic4.zhimg.com/80/v2-bed523d118d4c7178268e82e2bbce6db_720w.webp)

总的来说，资源消耗降低的前提下搜索和写入数据效率都有大幅度提升。下面就详细介绍Elasticsearch对地理信息索引的思路。

# 3.1 史前时代

Elasticsearch是基于Lucene构建的搜索引擎。Lucene最开始的设想是一个全文检索工具箱，即支持字符串检索，并没有考虑数值类型的处理。其核心思想非常简单，将文档分词后，为每个词构建一个term => array[docIds]的映射。

这样用户输入关键词只需要三步就可以获得想要的结果：

> 第一步: 通过关键词找到对应的倒排表。这一步简单来说就是查词典。例如：TermQuery.TermWeight 获取该term的倒排表，读取docId+freq信息。  
> 第二步: 根据倒排表得到的docId和词频信息对文档进行打分，返回给用户分值最高的TopN结果。例如：TopScoreDocCollector -- collect()方法，基于小顶堆，保留分数最大的TopN文档。  
> 第三步: 基于docId查询正排表获取文档字段明细信息。

这三步看起来简单，但简直是数据结构应用最佳战场，它需要综合考虑磁盘、内存、IO、数据结构时间复杂度，非常具有挑战性。

> 例如：查词典可以用很多数据结构实现，比如跳跃表，平衡树、HashMap等，而Lucene的核心工程师Mike McCandless实现了一个只有他自己能懂的FST, 是综合了有限自动机和前缀树的一种数据结构，用来平衡查询复杂度和存储空间，比HashMap慢，但是空间消耗低。文档打分通常用小顶堆来维护分值最高的N个结果，如果有新的文档打分超过堆顶，则替换堆顶元素即可。

**问题：**对于真实业务场景而言，只有字符串匹配查询是不够的，字符串和数值是应用最广泛的两种数据类型。如果需要进行区间查询怎么办呢？这是一个数据库产品非常基础的能力。

Lucene提供了一种适配方案RangeQuery。就是用枚举来模拟数值查询。简单来说：RangeQuery=BooleanQuery+TermQuery，所以限制查询是整数且区间最大不能超过1024。这种实现是可以说是非常鸡肋的，好在Lucene 2.9.0版本真正支持数值查询。

LUCENE-1470,LUCENE-1582,LUCENE-1602,LUCENE-1673,LUCENE-1701, LUCENE-1712

> Added NumericRangeQuery and NumericRangeFilter, a fast alternative to RangeQuery/RangeFilter for numeric searches. They depend on a specific structure of terms in the index that can be created by indexing using the new NumericField or NumericTokenStream classes. NumericField can only be used for indexing and optionally stores the values as string representation in the doc store. Documents returned from IndexReader/IndexSearcher will return only the String value using the standard Fieldable interface. NumericFields can be sorted on and loaded into the FieldCache. (Uwe Schindler, Yonik Seeley, Mike McCandless)  

这个实现很强大，支持了int/long/float/double/short/byte，也不限制查询区间了。它的核心思路是将数值字节数组化，然后利用前缀分层管理区间。

如下图所示：

![](https://pic1.zhimg.com/80/v2-f73ea8564e3ca0217d88aa26b39411f8_720w.webp)

本质上还是RangeQuery=BooleanQuery+TermQuery，只不过在前面做了一层转换：通过前缀树管理一个区间实现了匹配词数量的缩减，而这个缩减是非常有效的。所以这里就有一个专家参数：precisionStep。就是用来控制每个数值字段在分词是生成term的数量，生成term数量越多，区间控制粒度越细，占用磁盘空间越大，查询效率通常越高。

> 例如：如果precisionStep=8,则意味前缀树叶子节点的上层控制着255个叶子。那么，当查询范围在1~511时，由于跨了相邻的2个非叶子节点，所以需要遍历511个term。但是假如查询范围在0~512，又只需遍历2个term即可。这样的实现用起来真的有过山车的感觉。

综上，Elasticsearch核心的Lucene倒排索引是一种经典的以不变应万变：字符串和数值索引核心都是查倒排表。理解这个核心，对于后面理解地理位置数据存储和查询非常关键。接下来我们以geo_distance的实现思路为探索主线条，探索一下ES各个版本的实现思路。

# 3.2 Elasticsearch 2.0 版本

这个版本实现geo_distance查询的思路非常朴素，是建立在数值区间查询(NumericRangeQuery)的基础上。它的geo_point类型字段其实是一个复合字段，或者说是一个结构体。在底层实现时分别用两个独立字段索引来避免暴力扫描。即Elasticsearch的geo_point字段在实现上是lat,lon，加上编码成的geohash综合提供检索聚合功能。

字段定义如下所示：

```java
  public static final class GeoPointFieldType extends MappedFieldType {
 
        private MappedFieldType geohashFieldType;
        private int geohashPrecision;
        private boolean geohashPrefixEnabled;
 
        private MappedFieldType latFieldType;
        private MappedFieldType lonFieldType;
 
        public GeoPointFieldType() {}
}
```

算法的执行分为三个阶段：

**第一步：**根据中心点以及半径计算出一个大致符合需求的矩形区域，然后利用矩形区域的**最小最大经度得到一个数值区间查询**，利用矩形区域的**最小最大纬度得到一个区间查询**。

核心代码如下图所示：

```java
// 计算经纬度坐标+距离得到的矩形区域
// GeoDistance类
public static DistanceBoundingCheck distanceBoundingCheck(double sourceLatitude, double sourceLongitude, double distance, DistanceUnit unit) {
     // angular distance in radians on a great circle
     // assume worst-case: use the minor axis
     double radDist = unit.toMeters(distance) / GeoUtils.EARTH_SEMI_MINOR_AXIS;
 
     double radLat = Math.toRadians(sourceLatitude);
     double radLon = Math.toRadians(sourceLongitude);
 
     double minLat = radLat - radDist;
     double maxLat = radLat + radDist;
 
     double minLon, maxLon;
     if (minLat > MIN_LAT && maxLat < MAX_LAT) {
         double deltaLon = Math.asin(Math.sin(radDist) / Math.cos(radLat));
         minLon = radLon - deltaLon;
         if (minLon < MIN_LON) minLon += 2d * Math.PI;
         maxLon = radLon + deltaLon;
         if (maxLon > MAX_LON) maxLon -= 2d * Math.PI;
     } else {
         // a pole is within the distance
         minLat = Math.max(minLat, MIN_LAT);
         maxLat = Math.min(maxLat, MAX_LAT);
         minLon = MIN_LON;
         maxLon = MAX_LON;
     }
 
     GeoPoint topLeft = new GeoPoint(Math.toDegrees(maxLat), Math.toDegrees(minLon));
     GeoPoint bottomRight = new GeoPoint(Math.toDegrees(minLat), Math.toDegrees(maxLon));
     if (minLon > maxLon) {
         return new Meridian180DistanceBoundingCheck(topLeft, bottomRight);
     }
     return new SimpleDistanceBoundingCheck(topLeft, bottomRight);
 }
```

**第二步：**两个查询通过BooleanQuery组合成一个取交集的复合查询，以实现初筛出在经纬度所示矩形区域内的docId集合。

核心代码如下图所示：

```java
public class IndexedGeoBoundingBoxQuery {
 
    public static Query create(GeoPoint topLeft, GeoPoint bottomRight, GeoPointFieldMapper.GeoPointFieldType fieldType) {
        if (!fieldType.isLatLonEnabled()) {
            throw new IllegalArgumentException("lat/lon is not enabled (indexed) for field [" + fieldType.names().fullName() + "], can't use indexed filter on it");
        }
        //checks to see if bounding box crosses 180 degrees
        if (topLeft.lon() > bottomRight.lon()) {
            return westGeoBoundingBoxFilter(topLeft, bottomRight, fieldType);
        } else {
            return eastGeoBoundingBoxFilter(topLeft, bottomRight, fieldType);
        }
    }
 
    private static Query westGeoBoundingBoxFilter(GeoPoint topLeft, GeoPoint bottomRight, GeoPointFieldMapper.GeoPointFieldType fieldType) {
        BooleanQuery.Builder filter = new BooleanQuery.Builder();
        filter.setMinimumNumberShouldMatch(1);
        filter.add(fieldType.lonFieldType().rangeQuery(null, bottomRight.lon(), true, true), Occur.SHOULD);
        filter.add(fieldType.lonFieldType().rangeQuery(topLeft.lon(), null, true, true), Occur.SHOULD);
        filter.add(fieldType.latFieldType().rangeQuery(bottomRight.lat(), topLeft.lat(), true, true), Occur.MUST);
        return new ConstantScoreQuery(filter.build());
    }
 
    private static Query eastGeoBoundingBoxFilter(GeoPoint topLeft, GeoPoint bottomRight, GeoPointFieldMapper.GeoPointFieldType fieldType) {
        BooleanQuery.Builder filter = new BooleanQuery.Builder();
        filter.add(fieldType.lonFieldType().rangeQuery(topLeft.lon(), bottomRight.lon(), true, true), Occur.MUST);
        filter.add(fieldType.latFieldType().rangeQuery(bottomRight.lat(), topLeft.lat(), true, true), Occur.MUST);
        return new ConstantScoreQuery(filter.build());
    }
}
```

**第三步：**利用FieldData缓存(正向信息)根据docId获取矩形区域中每个坐标点的经纬度，然后利用前面的Haversine公式计算跟中心坐标点的距离，进行精确筛选，得到符合条件的文档集合。

核心代码如下所示：

```java
// GeoDistanceRangeQuery类的实现
 @Override
 public Weight createWeight(IndexSearcher searcher, boolean needsScores) throws IOException {
     final Weight boundingBoxWeight;
     if (boundingBoxFilter != null) {
         boundingBoxWeight = searcher.createNormalizedWeight(boundingBoxFilter, false);
     } else {
         boundingBoxWeight = null;
     }
     return new ConstantScoreWeight(this) {
         @Override
         public Scorer scorer(LeafReaderContext context) throws IOException {
             final DocIdSetIterator approximation;
             if (boundingBoxWeight != null) {
                 approximation = boundingBoxWeight.scorer(context);
             } else {
                 approximation = DocIdSetIterator.all(context.reader().maxDoc());
             }
             if (approximation == null) {
                 // if the approximation does not match anything, we're done
                 return null;
             }
             final MultiGeoPointValues values = indexFieldData.load(context).getGeoPointValues();
             final TwoPhaseIterator twoPhaseIterator = new TwoPhaseIterator(approximation) {
                 @Override
                 public boolean matches() throws IOException {
                     final int doc = approximation.docID();
                     values.setDocument(doc);
                     final int length = values.count();
                     for (int i = 0; i < length; i++) {
                         GeoPoint point = values.valueAt(i);
                         if (distanceBoundingCheck.isWithin(point.lat(), point.lon())) {
                             double d = fixedSourceDistance.calculate(point.lat(), point.lon());
                             if (d >= inclusiveLowerPoint && d <= inclusiveUpperPoint) {
                                 return true;
                             }
                         }
                     }
                     return false;
                 }
             };
             return new ConstantScoreScorer(this, score(), twoPhaseIterator);
         }
     };
 }
```

这是一种非常简单且直观的思路实现了中心点指定半径范围POI的搜索能力。

简单总结一下要点：

1.  利用中心点坐标和半径确定矩形区域边界。  
    
2.  利用Bool查询综合两个NumericRangeQuery查询，实现矩形区域初筛。  
    
3.  利用Haversine公式计算中心点和矩形区域内每个坐标点距离，进行第二阶段过滤操作，筛选出最终符合条件的docId集合。

方案虽然简单，但是毕竟实现了geo_distance的能力。又不是不能用，对吧？那么该方案有什么问题呢？

# 3.3 Elasticsearch 2.2 版本

ES2.0版本的实现有个问题， 就是没有很好解决二维组合条件查询的数据筛选。它是分别获取符合纬度范围条件的文档集合和符合经度范围条件的文档集合然后进行交集，初筛了太多无效的文档集合。

它的处理思路用一张图表示如下：

![](https://pic3.zhimg.com/80/v2-c8bc07e23ddc8a48986ec704423693aa_720w.webp)

即选择了那么多的记录，最终只有经纬度范围交汇的红色区域是初筛的范围。

针对上面的问题，ES 2.2版本引入特性：基于四叉树(Quadtree)的地理位置查询(Lucene 5.3版本实现)。

Quadtree并非什么复杂高深的数据结构，相比二叉树，多了两个子节点。

作为一种基础的数据结构，Quadtree应用场景非常广泛，在图像处理、空间索引、碰撞检测、人生游戏模拟、分形图像分析等领域都可以看到它的身影。

在Elasticsearch地理位置空间索引问题上，Quadtree用来表示区间，可以视为前缀树的一种。

-   **Region quadtree**

> The region quadtree represents a partition of space in two dimensions by decomposing the region into four equal quadrants, subquadrants, and so on with each leaf node containing data corresponding to a specific subregion. Each node in the tree either has exactly four children, or has no children (a leaf node). The height of quadtrees that follow this decomposition strategy (i.e. subdividing subquadrants as long as there is interesting data in the subquadrant for which more refinement is desired) is sensitive to and dependent on the spatial distribution of interesting areas in the space being decomposed. The region quadtree is a type of trie.  

在区间划分上，Quadtree跟geohash的处理思路有些相似。在一维世界，二分可以无限迭代。同理，在二维世界，四分也可以无限迭代。下面这个图可以非常形象展示Quadtree的区间划分过程。

![](https://pic2.zhimg.com/80/v2-2a259745360e833fa34c198e193e65b9_720w.webp)

**ES 2.2是如何使用Quadtree来实现geo_distance查询呢？**

通常我们使用一种数据结构，是先基于该数据结构存储数据，然后查询这个数据结构。ES这里使用Quadtree的做法非常巧妙：存储的时候没有感觉用到Quadtree，查询时却用其查询方式。

**morton编码：**在理解ES的处理思路前，需要科普一个知识点，那就是morton编码。关于morton编码，跟geohash类似，是一种将二维数据按二进制位交叉编码成一维数据的一种网格编码，其用法和特点跟geohash也是类似的。对于64位的morton码，其经纬度定位精度范围控制到了厘米级别，对于地理位置场景而言，是非常非常高的精度了。

**数据存储：**ES2.2版本之前一个经纬度坐标需要三个字段存储：lat,lon,geohash。有了Quadtree后，只需要一个字段存储就可以了。具体的实现思路如下：将lat,lon坐标进行映射，使得经纬度的取值范围从[-180,180]/[-90,90]映射到[0,2147483520](整数好处理)， 然后处理成一维的mortonHash数值。对于数值字段的处理思路，就又回到了前缀(trie)的思路，就又回到了熟悉的专家参数precisionStep。在这里的前缀该如何理解？对于一维数据，每个前缀管理一段区间，对于二维数据每个前缀管理一个二维网格区域。例如一个坐标点利用precisionStep=9来划分前缀，其可视化矩形区域如下：

(取shift=27,36)

![](https://pic3.zhimg.com/80/v2-3ae12abb4283341b5775242502961b8a_720w.webp)

(取shift=36,45)

![](https://pic3.zhimg.com/80/v2-a847c96f4e78d422d666a14bd7ac56ba_720w.webp)

**数据查询：**在查询时，首先将查询中心点坐标转换成一个矩形。这个处理思路我们延续了ES 2.0的做法，不陌生了。

例如：对于坐标为(116.433322,39.900255)，半径为1km的点，生成的矩形如下所示：

```java
double centerLon = 116.433322;
double centerLat = 39.900255;
double radiusMeters = 1000.0;
GeoRect geoRect = GeoUtils.circleToBBox(centerLon, centerLat, radiusMeters);
System.out.println( geoRect );
```

用高德API生成对应的可视化图形如下：

![](https://pic3.zhimg.com/80/v2-666ae3756a97f62f01b18783c8dae02a_720w.webp)

有了这个矩形后，后面的做法就跟ES 2.0有些不同了。ES 2.2版本的思路是利用Quadtree对**整个世界**地图进行网格化。具体的流程如下：

-   **Quadtree处理流程**

> **第一步:** 以经纬度(0,0)为起始中心点，将整个世界切分成4个区块。并判断参数生成的矩形在哪个区块。  
> **第二步:** 对于矩形区域不在的区域，略过。对于矩形区域所在的区块，继续四分，切成4个区块。  
> **第三步：** 当满足如下任一条件时，将相关的文档集合收集起来，作为第一批粗筛的结果。

-   条件一：切分到正好跟前缀的precisionStep契合，并且quad-cell在矩形内部时。
-   条件二：切分到最小层级(level=13)时且quad-cell跟矩形区域有交集时。

**第四步:** 利用lucene的doc_values缓存机制，获取每个docId对应的经纬度，利用距离公式计算是否在半径范围内，得到最终的结果。(这个操作也是常规思路了)

![](https://pic2.zhimg.com/80/v2-034351d99ec1d31536969bf158ecb621_720w.webp)

另外ES在处理时进行了版本兼容。

> 例如：ES 2.2版本对于geo_distance的实现关键点，判断索引版本是否是V_2_2_0版本以后创建，如果是则直接用Lucene的GeoPointDistanceQuery查询类，否则沿用ES 2.0版本的GeoDistanceRangeQuery。

```java
 IndexGeoPointFieldData indexFieldData = parseContext.getForField(fieldType);
final Query query;
if (parseContext.indexVersionCreated().before(Version.V_2_2_0)) {
    query = new GeoDistanceRangeQuery(point, null, distance, true, false, geoDistance, geoFieldType, indexFieldData, optimizeBbox);
} else {
    distance = GeoUtils.maxRadialDistance(point, distance);
    query = new GeoPointDistanceQuery(indexFieldData.getFieldNames().indexName(), point.lon(), point.lat(), distance);
}
 
if (queryName != null) {
    parseContext.addNamedQuery(queryName, query);
}
```

> **核心代码参考：**GeoPointDistanceQuery、GeoPointRadiusTermsEnum

# 3.4 Elasticsearch 5.0 版本

方案优化的探索是没有没有止境的，Lucene的核心工程师 Michael McCandless受到论文《Bkd-Tree: A Dynamic Scalable kd-Tree》启发，基于BKD tree再次升级了地理位置数据索引建模和查询功能。

这个数据结构不仅仅是用于解决地理位置查询问题，更是数值类数据索引建模的通用方案。它可以处理一维的数值，从byte到BigDecimal, IPv6地址等等；它也可以处理二维乃至于N维的数据检索问题。

-   **LUCENE-6825**

> This can be used for very fast 1D range filtering for numerics, removing the 8 byte (long/double) limit we have today, so e.g. we could efficiently support BigInteger, BigDecimal, IPv6 addresses, etc.  
> It can also be used for > 1D use cases, like 2D (lat/lon) and 3D (x/y/z with geo3d) geo shape intersection searches.  
> ...  
> It should give sizable performance gains (smaller index, faster searching) over what we have today, and even over what auto-prefix with efficient numeric terms would do.

在前面的版本中，对于数值区间查询的处理思路本质上都是term匹配，通过前缀实现了一个term管理一个区间，从而降低了区间查询需要遍历的term数量。而从ES 5.0版本开始，彻底优化了数值查询(从一维到N维)，其底层是Lucene 6.0版本实现的BKD tree的独立索引。其实现不仅降低了内存开销，而且提升了检索和索引速度。

关于bkd-tree的原理，其大体思路如下。在面对数值查询区间查询的问题上，大体分为两个层次：

> 【优化内存查询】：BST(binary-search-tree) > Self-balanced BST > kd-tree。  
> 【优化外存(硬盘)查询】：B-tree > K-D-B-tree > BKD tree。

kd-tree其实就是多维的BST。例如：

![](https://pic1.zhimg.com/80/v2-eaccd3ea2b57c3fe3932b001b843b868_720w.webp)

**【数据存储】：**BKD tree的核心思路是非常简单的，将N维点集合形成的矩形空间(southWest,northEast)递归分割成更小的矩形空间。跟常见的kd-tree不同，当分割到网格区域里面坐标点的数量小于一定数量(比如1024)就停止了。

例如：

![](https://pic3.zhimg.com/80/v2-597ea3d004339d8c92604af9c1e9f5fe_720w.webp)

通过区域的分割，确保每个区域POI的数量大致相等。

**【数据查询】：**搜索的时候，就不再是像Quadtree从整个世界开始定位，而是基于当前的点集合形成的空间来查找。例如以geo_distance查询为例。

其流程如下：

**第一步:** 中心点坐标+半径生成一个矩形(shape boundary)。这一步是常规操作了，前面的版本也都是这么做的。

**第二步：**该矩形跟BKD tree 叶子节点形成的矩形(cell)进行intersect运算，所谓intersect运算，就是计算两个矩形的位置关系：相交、内嵌还是不相关。query和bkd-tree形成的区域有三种关系。

![](https://pic3.zhimg.com/80/v2-dfcc229a7038c91bc0fe6595483624d2_720w.webp)

对于CELL_CROSSES_QUERY，如果是叶子节点，则需要判断cell中的每个POI是否符合query的查询条件；否则查询子区间；对于CELL_OUTSIDE_QUERY，直接略过；对于CELL_INSIDE_QUERY，整个cell中的POI都满足查询条件。

> 核心代码：LatLonPoint/LatLonPointDistanceQuery

# 3.5 后续发展

Geo查询能力的迭代和变迁，其实也是Elasticsearch作为一个数据库对数值查询能力的升级和优化，扩展产品的适用场景，让使用者打破对Elasticsearch只能做全文检索的偏见。从全文检索数据库扩展到分析型数据库，Elasticsearch还有很长的路要走。

按照 Michael McCandless的设想，当前的多维数据只能是单个点，但是有些场景需要将形状作为一个维度进行索引。在这种需求下，需要通过一种更普适化的k-d tree ，即R-Tree来实现。

路漫漫其修远兮，ES从2.0版本支持geo-spatial开始经历6年的发展，已经走了很远，然而依然有很多值得探索的领域和场景。

# Reference
https://zhuanlan.zhihu.com/p/532876904