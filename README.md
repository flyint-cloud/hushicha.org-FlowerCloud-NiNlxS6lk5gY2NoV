
**大纲**


**1\.社区电商的业务闭环**


**2\.Redis缓存架构的典型生产问题**


**3\.用户数据在读多写少场景下的缓存设计**


**4\.热门用户数据的缓存自动延期机制**


**5\.缓存惊群与穿透问题的解决方案**


**6\.缓存和数据库双写不一致问题分析**


**7\.基于分布式锁保证缓存和数据库双写一致性**


**8\.缓存和数据库双写在分布式锁高并发下的优化**


**9\.利用分布式锁自动超时消除串行等待锁的影响**


**10\.写少读多的企业级缓存架构设计总结**


 


**1\.社区电商的业务闭环**


接下来介绍的社区电商是以Redis作为主体技术、以MySQL和RocketMQ作为辅助技术实现的。


 


**(1\)社区电商运作模式**


社区电商的关键点在于社区，而电商则是辅助性质(次要地位，流量变现)。社区可以分成很多种社区，比如美食社区、美妆社区、影评社区、妈妈社区。社区平台也有很多很多，一般中小型的社区平台居多，比如体育社区、汽车社区、本地生活社区。


 


比如美食社区APP：用户可以分享积累的美食食谱、或者对美食看法、甚至是出门体验的一些餐馆，用户还可以浏览其他用户发出的一些美食食谱、体验、经历、科普。用户通过浏览其他用户发的帖子，对其进行互动、关注、私信、交流、成为好友。这样一部分用户就可以成立平台里的一个私密圈子，基于美食兴趣爱好进行社交活动。


 


美食社区APP里会有一些用户发出的帖子内容特别优质，这些帖子内容会吸引很多用户来浏览，浏览量可能会非常大。这时可以在这些帖子添加一些推荐商品，这样浏览帖子的用户就会看到推荐的商品，可以点击商品链接，进入商品详情页，发生购物行为。


 


**(2\)电商APP的feed流**


feed流指的是APP不断地、主动地显示各种新内容给用户，如果用户主动搜索和浏览就不是feed流。


 


用户在电商APP首页不断进行下拉时：电商APP会根据用户的喜好、爆款，通过算法不停地显示一批新的内容给用户，这种展示商品的方式就是电商APP的feed流。


 


比如当用户进入社区电商APP的首页后，进行不停下拉时：电商APP会把用户关注过的大v、可能感兴趣的美食帖子、浏览量高的爆款帖子，通过算法不停地计算出新的一批内容显示给用户， 这就社区电商的feed流。


 


此外，用户还可以在社区电商APP根据条件和分页进行结构化查询。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/12b34afbb63d453baf471b7176da5c7c~tplv-obj.image?lk3s=ef143cfe&traceid=202412131912159F380F2E8AD80435A24B&x-expires=2147483647&x-signature=VsOwwEmwggkx%2BBbKn%2FK%2Fsehd2lY%3D)
**(3\)社区电商的流量变现交易闭环**


对于小红书这些社区电商APP来说，当社区互动做好了之后，APP就会吸引大量流量。而对于这种有大量流量的社区APP，流量变现的最好模式就是种草。


 


所谓种草就是用户发布分享帖子时，可以在分享内容里插入一些商品推荐并给出商品链接。这样当其他用户在浏览这些分享帖子时，就会看到推荐的商品和链接。然后点击链接就可以进入商品详情页，查看商品标题、图文视频介绍、价格、营销、库存等。接着加入购物车并发起订单提交、支付、履约，最后就能拿到商品。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/d43e5ad794244b05875225db5e65cc81~tplv-obj.image?lk3s=ef143cfe&traceid=202412131912159F380F2E8AD80435A24B&x-expires=2147483647&x-signature=5N%2B2xugpeNCFksjnp1Cj4VM0XcU%3D)
接下来主要介绍社区电商部分功能点的实现：首页feed流、帖子分享浏览详情、社交分享和团购、商品详情和库存、购物车，这些功能会基于Redis的企业级缓存方案(主要) \+ RocketMQ(部分)来实现。


 


**2\.Redis缓存架构的典型生产问题**


Redis的典型生产问题如下：


 


**问题一：热key问题**


热key就是某个key形成了热点。比如某明星突然官宣离婚，那么就会出现大量用户瞬时涌入该明星微博进行围观的情况。从而出现瞬时百万级千万级请求去获取Redis某个key的数据，这就是热key问题。


 


对于社区电商APP来说，如果有一个比较好的帖子分享和团购活动，那么也有可能短时间内引发大量用户把这该帖子详情页分享到微信等社交应用。从而引发大量用户在短时间内查看该分享详情页，最后造成Redis热key问题。


 


**问题二：大value问题**


存储的key\-value特别大，比如value多达10M。这个value如果被频繁读取，那么就有可能把Redis机器的网络带宽打满，阻塞别的请求。


 


**问题三：缓存穿透击穿问题**


缓存穿透是指缓存和数据库中都没有的数据，而用户不断发起请求(布隆过滤器或设置空对象)。缓存击穿是指缓存中没有但数据库中有的数据(一般是缓存时间到期)，这时由于并发请求特别多，同时读缓存没读到数据，又同时去数据库去取数据。


 


**问题四：缓存失效和LRU被清理的问题**


缓存数据设置了过期的时间，到期失效后应该如何来处理。Redis如果内存满了，LRU算法会自动淘汰一些数据，对于这些数据应该如何进行处理，如何才能实现自动加载和重建。


 


**问题五：缓存雪崩问题**


缓存雪崩是指缓存中数据大批量到过期时间，而查询量巨大，引起数据库压力过大甚至宕机。和缓存击穿不同的是：缓存击穿是指并发查同一条数据，缓存雪崩是不同数据都过期了。


 


如果Redis集群都崩掉了，只有数据库可以访问。那么首先就需要自动识别出缓存故障，然后马上进行限流对数据库进行保护，不让数据库崩溃，以及马上启动各个接口的降级机制。


 


各个接口的降级机制可以提前在JVM内存里，准备少量缓存作为降级备用数据。所以每个接口都需要有一个降级方案，一旦出现缓存故障，那么就可以自动限流避免数据库崩溃。


 


限流 \-\> 降级 \-\> 把JVM内存里缓存的默认数据给用户或者 直接对用户进行提醒。


 


**问题六：数据库的一致性问题**


缓存数据和数据库之间的一致性的保障，双写、异步同步如何保证一致性。


 


**Redis生产总结：**


Redis上了生产以后，首先需要模拟出足量的数据写入Redis里，比如模拟出千万级数据量写入部署好的Redis集群中。然后进行高并发压测，并通过CacheCloud进行监控运维。监控出有多少个缓存节点、里面放了多少G数据、大压力下接口性能如何、QPS多少、Redis机器负载如何、缓存命中率如何、数据库回源比例是多少、演示Redis节点故障的主从切换、演示Redis集群扩容等。


 


**3\.用户数据在读多写少场景下的缓存设计**


**具体的缓存设计如下：**


一.新增或更新用户时先获取分布式锁，避免短时间发生多次请求出现重复新增或更新


二.用户数据会先写数据库，再写Redis缓存


三.用户数据属于读多写少场景下的数据，适合用缓存支持高并发场景下的读取


四.由于大部分用户数据属于冷门数据，故其缓存的过期时间设置为2天加随机几小时


五.如果后面有请求需要频繁访问某条用户数据，那么可以不断延长(重置)其缓存的过期时间


 


**具体的代码如下：**



```
@Override
public SaveOrUpdateUserDTO saveOrUpdateUser(SaveOrUpdateUserRequest request) {
    //新增或更新用户时先需要获取分布式锁，避免短时间发生多次请求出现重复新增或更新，保证幂等性
    String userUpdateLockKey = RedisKeyConstants.USER_UPDATE_LOCK_PREFIX + request.getOperator();
    boolean lock = redisLock.lock(userUpdateLockKey);

    if (!lock) {
        log.info("获取锁失败，operator:{}", request.getOperator());
        throw new BaseBizException("新增/修改失败");
    }

    try {
        //用户数据先写入数据库
        CookbookUserDO cookbookUserDO = cookbookUserConverter.convertCookbookUserDO(request);
        cookbookUserDO.setUpdateUser(request.getOperator());
        if (Objects.isNull(cookbookUserDO.getId())) {
            cookbookUserDO.setCreateUser(request.getOperator());
        }
        cookbookUserDAO.saveOrUpdate(cookbookUserDO);
        CookbookUserDTO cookbookUserDTO = cookbookUserConverter.convertCookbookUserDTO(cookbookUserDO);

        //用户数据写入数据库后，再写入Redis缓存，并生成随机的过期时间
        //每条用户数据都会在Redis里缓存2天加上随机几小时
        //这样后续出现高并发读取用户数据时，就可以直接从Redis缓存里获取用户数据了
        //而且用户数据一般是不会变化的，属于读多写少场景下的数据
        //像这种读多写少场景下的数据，就非常适合用缓存来支持高并发场景下的读取

        //此外，由于大部分用户数据不会被经常读取，所以可以设置默认来2天多就可以过期了
        //如果后面有请求需要访问这条用户数据，但在缓存里没找到，那么再从数据库里加载出写入缓存即可
        //如果后面有请求需要频繁访问这条用户数据，那么可以不断延长其缓存的过期时间
        redisCache.set(RedisKeyConstants.USER_INFO_PREFIX + cookbookUserDO.getId(),
            JsonUtil.object2Json(cookbookUserDTO), CacheSupport.generateCacheExpireSecond());

        SaveOrUpdateUserDTO dto = SaveOrUpdateUserDTO.builder().success(true).build();
        return dto;
    } finally {
        redisLock.unlock(userUpdateLockKey);
    }
}

public interface CacheSupport {
    Integer TWO_DAYS_SECONDS = 2 * 24 * 60 * 60;
    //生成缓存过期时间：2天加上随机几小时
    static Integer generateCacheExpireSecond() {
        return TWO_DAYS_SECONDS + RandomUtil.genRandomInt(0, 10) * 60 * 60;
    }
}
```

 


**4\.热门用户数据的缓存自动延期机制**


由于用户数据是属于读多写少场景下的数据，所以在新增或更新时对其进行缓存是很合适的。进行缓存后，在其他各种场景下，获取用户数据时就可以直接从缓存里进行读取。


 


为什么对用户数据进行缓存时，过期时间被设定为：2天加上随机几小时？


 


因为只有少数的用户数据是热门数据，这些用户发表的分享会占据绝大部分的浏览量。而大部分的用户数据都是冷门数据，这些用户发表的分享几乎没有多少浏览量。因此没有必要让所有的用户数据都驻留在缓存里，占用缓存宝贵的内存空间。


 


如果缓存好的某用户数据后续没有被访问，那么就让它过期即可。当过一段时间后有请求需要访问该用户数据时，再重新回源数据库进行查询并写入缓存。


 


如果缓存好的一条用户数据后续被不断请求访问，成为了热门用户数据。那么每次访问该用户数据，可以延长(重置)其缓存的过期时间。这样就可以从热门数据一直都可以从缓存里读取，实现高并发读。


 


注意：从缓存里获取不到用户数据，需要回源数据库进行查询并写入缓存时，首先要加分布式锁。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/d1d745a1ece64c80957b0d83e81b8e27~tplv-obj.image?lk3s=ef143cfe&traceid=202412131912159F380F2E8AD80435A24B&x-expires=2147483647&x-signature=ChCPUNg2sGhEMhWMRGAn6XG1EQA%3D)

```
@Override
public CookbookUserDTO getUserInfo(CookbookUserQueryRequest request) {
    Long userId = request.getUserId();
    //先从缓存中尝试获取用户数据
    CookbookUserDTO user = getUserFromCache(userId);
    if (user != null) {
        return user;
    }
    //回源数据库进行读取
    return getUserInfoFromDB(userId);
}

private CookbookUserDTO getUserFromCache(Long userId) {
    String userInfoKey = RedisKeyConstants.USER_INFO_PREFIX + userId;
    String userInfoJsonString = redisCache.get(userInfoKey);
    log.info("从缓存中获取用户信息,userId:{},value:{}", userId, userInfoJsonString);

    if (StringUtils.hasLength(userInfoJsonString)) {
        //通过设置空值来防止缓存穿透
        if (Objects.equals(CacheSupport.EMPTY_CACHE, userInfoJsonString)) {
            return new CookbookUserDTO();
        }
        //每次有读请求，就自动延长过期时间
        redisCache.expire(RedisKeyConstants.USER_INFO_PREFIX + userId, CacheSupport.generateCacheExpireSecond());
        CookbookUserDTO dto = JsonUtil.json2Object(userInfoJsonString, CookbookUserDTO.class);
        return dto;
    }

    return null;
}
```

 


**5\.缓存惊群与穿透问题的解决方案**


**(1\)用户数据写入和查询时的缓存设计总结**


**(2\)设置过期时间为2天加随机几小时的原因**


**(3\)缓存穿透问题的解决方案**


 


**(1\)用户数据写入和查询时的缓存设计总结**


写入用户数据时，会对数据库和缓存进行双写，缓存默认2天多随机时间过期。如果用户数据被频繁读取，那么其缓存就会不停地自动延期(重置)。如果用户数据没被读取，那么就按默认设定自动过期，避免占用缓存空间。当用户数据不能从缓存中读取到，则先从数据库里查出来，然后再放入缓存里。


 


**(2\)设置过期时间为2天加随机几小时的原因**


随机几小时是为了避免缓存惊群问题。如果缓存的一批数据的过期时间都设置一样，那么就会出现大量缓存同时过期的情况。这会造成大量的瞬时请求去访问MySQL，对MySQL造成压力。为了避免该问题，可以设置缓存数据的过期时间都是随机的，不集中在某个时间点一起过期。


 


惊群效应是技术里的术语，指的是突然在某个时间点出了一个故障，导致一大片范围线程、进程、机器都同时被惊动了。


 


**(3\)缓存穿透问题的解决方案**


缓存穿透是指缓存和数据库中都没有的数据，而用户不断发起请求(布隆过滤器或设置空对象)。如果大量请求的是同一批key(缓存和DB都没数据)，则可以对这些key缓存一个空对象。如果大量请求的是不同的key(缓存和DB都没数据)，则可以使用布隆过滤器过滤这些key。所以对从DB查出空数据的key缓存空对象，也不一定完全解决缓存穿透问题。


 


使用布隆过滤器过滤key时，应该是对DB里的所有数据进行添加，在查询缓存前过滤。但是如果在查询缓存前对所有key进行过滤(高风险方案)，那么就存在很大风险。因为如果过滤所有key就发生了故障，那么可能会导致所有数据都被误判为没有数据。


 


所以可以考虑默认采用设置空对象来避免缓存穿透，然后统计这种缓存穿透对DB的请求数。如果请求数超出一定阈值，则说明有大量请求都是不同的key了，设置空对象无法避免，此时再升级为布隆过滤器来避免这种缓存穿透。


 


**6\.缓存和数据库双写不一致问题分析**


**(1\)对缓存和数据库双写时发生不一致的场景**


**(2\)缓存和数据库双写时不一致问题的解决方案**


 


**(1\)对缓存和数据库双写时发生不一致的场景**


某用户数据在Redis的缓存已经过期了，此时刚好有两个线程分别并发去对给用户数据进行读取和更新。第一个线程在读取时，发现缓存没有数据，于是就去读库，读完库后会更新缓存。第二个线程在更新时，会先更新数据库，然后再更新缓存。以上两个线程也刚好对应了写缓存的两个场景。


 


由于这两个线程并发执行，那么就可能出现如下产生不一致的场景：第一个线程首先读库读到了旧值，还没来得及将读到的旧值写入缓存时。第二个线程的新值更新已经完成了写库和写缓存，此时缓存数据是最新的。接着才轮到第一个线程进行写缓存，但是这时候写的数据却是一开始读到的旧值。于是第一个线程写缓存时就把缓存里的最新数据给覆盖了，从而出现数据库里的是新数据，但缓存里的是旧数据，产生了不一致。


 


**(2\)缓存和数据库双写时不一致问题的解决方案**


有一个简单易行的方案来解决这个问题，就是使用分布式锁让数据库读和写必须是串行化，所以接下来可以对数据库进行读和写时加同一个分布式锁。


 


**7\.基于分布式锁保证缓存和数据库双写一致性**


用户数据是典型读多写少的数据，可能0\.01%是写，99\.99%是读，所以进行用户注册和信息更新的操作是极少的。社区平台平时对用户数据大部分都是进行查询，比如分享帖子的详情页、feed流页面就需要查询用户信息。


 


基于用户数据读多写少的特点，在保证缓存和数据库双写一致性时，就需要注意以下几点。


 


**注意一：不能在读缓存处加锁而在读库时加锁**


在用户数据写的地方加锁，由于用户数据具有读多写少的特点，所以几乎不会影响性能。在用户数据读的地方加锁，就需要注意不能在读缓存处加锁，而应该在准备读库时加锁。


 


**注意二：写库和读库加的锁是同一把锁**


写数据库的加的锁和准备读数据库的地方加的锁，是同一把锁。


 


**注意三：获取读库的锁后进行双重检查**


如果一个线程在获得了读数据库的锁之后，需要进行双重检查，避免再去数据库查一次。因为如果出现大量的请求并发读取某个已过期的用户数据缓存时，此时只会有一个线程获取到锁去查库，然后其他大量的线程都只能在串行化排队。当获取到锁的线程完成读库 \+ 更新缓存并释放锁后，其他线程就没必要再查库了，否则影响性能。



```
@Override
public CookbookUserDTO getUserInfo(CookbookUserQueryRequest request) {
    //由于用户数据具有读多写少的特点，在用户数据读的地方加锁时
    //需要注意不能在这里的读缓存处加锁，而应该在准备读库时加锁，否则会影响性能

    Long userId = request.getUserId();
    //先从缓存中尝试获取用户数据
    CookbookUserDTO user = getUserFromCache(userId);
    if (user != null) {
        return user;
    }
    //回源数据库进行读取
    return getUserInfoFromDB(userId);
}

//从缓存中获取不到用户数据需要回源数据库
//首先会获取分布式锁，以防止并发出现两个线程：
//一个线程A在读某用户数据，一个线程B在更新该用户数据，但此时缓存里的该用户数据已过期
//读取该用户数据的线程A先从数据库获取到旧值，紧接着更新该用户数据的线程B马上完成数据库+缓存的新值更新
//之后读取该用户数据的线程A才执行到更新缓存这一步骤，于是使用了旧值去覆盖线程B更新好的新值
private CookbookUserDTO getUserInfoFromDB(Long userId) {
    //首先获取分布式锁，防止出现两个读写线程造成缓存和DB不一致，这个锁和更新用户数据时用来保证幂等性的锁是同一把锁
    String userLockKey = RedisKeyConstants.USER_UPDATE_LOCK_PREFIX + userId;
    boolean lock = false;
    try {
        //尝试加锁并且设置锁的超时时间
        //第一个拿到锁的线程在超时时间内处理完事情会释放锁，其他线程会继续竞争锁
        //而在这个超时时间里没有获得锁的线程会被挂起并进入队列进行串行等待
        //如果在这个超时时间外还获取不到锁，排队的线程就会被唤醒并返回false
        lock = redisLock.tryLock(userLockKey, USER_UPDATE_LOCK_TIMEOUT);
    } catch(InterruptedException e) {
        CookbookUserDTO user = getUserFromCache(userId);
        if (user != null) {
            return user;
        }
        log.error(e.getMessage(), e);
        throw new BaseBizException("查询失败");
    }

    //获取分布式锁超时失败
    if (!lock) {
        //并发进来串行排队的线程获取分布式锁超时返回失败后，就重新读缓存(实现"串行等待锁+读缓存"转"串行读缓存")
        CookbookUserDTO user = getUserFromCache(userId);
        if (user != null) {
            return user;
        }
        log.info("缓存数据为空，从数据库查询用户数据时获取锁失败，userId:{}", userId);
        throw new BaseBizException("查询失败");
    }

    //获取分布式锁成功
    try {
        //1.先读缓存，进行Double Check双重检查，防止并发进来等待锁释放的线程重复去读DB
        //第一个获取锁的请求设置好缓存后，第二个获取到锁的请求就不需要再去查DB了
        CookbookUserDTO user = getUserFromCache(userId);
        if (user != null) {
            return user;
        }
        log.info("缓存数据为空，从数据库中获取数据，userId:{}", userId);

        String userInfoKey = RedisKeyConstants.USER_INFO_PREFIX + userId;

        //2.再读DB
        //此时需要注意：
        //读取某用户数据的线程A首先在这里读到了DB里的旧值，还没来得及接着执行更新读取到的旧值到缓存
        //然后更新该用户数据的线程B就完成了更新新值的所有操作(更新新值到DB + 缓存)
        CookbookUserDO cookbookUserDO = cookbookUserDAO.getById(userId);
        if (Objects.isNull(cookbookUserDO)) {
            //如果从DB读取到了空数据，那么就对该key设置空值，避免缓存穿透
            redisCache.set(userInfoKey, CacheSupport.EMPTY_CACHE, CacheSupport.generateCachePenetrationExpireSecond());
            return null;
        }

        CookbookUserDTO dto = cookbookUserConverter.convertCookbookUserDTO(cookbookUserDO);

        //3.缓存数据，设置缓存时间为2天+随机几小时，避免缓存惊群
        //此时需要注意：
        //更新某用户数据的线程B已经把缓存里的该用户数据更新到最新了，
        //读取该用户数据的线程A才执行到这里开始把获取到的旧值更新到缓存，这样就产生数据库和缓存不一致了
        redisCache.set(userInfoKey, JsonUtil.object2Json(dto), CacheSupport.generateCacheExpireSecond());
        return dto;
    } finally {
        redisLock.unlock(userLockKey);
    }
}

@Override
public SaveOrUpdateUserDTO saveOrUpdateUser(SaveOrUpdateUserRequest request) {
    //新增或更新用户时先需要获取分布式锁，避免短时间发生多次请求出现重复新增或更新，保证幂等性
    String userUpdateLockKey = RedisKeyConstants.USER_UPDATE_LOCK_PREFIX + request.getOperator();
    boolean lock = redisLock.lock(userUpdateLockKey);

    if (!lock) {
        log.info("获取锁失败，operator:{}", request.getOperator());
        throw new BaseBizException("新增/修改失败");
    }
    ...
}
```

 


**8\.缓存和数据库双写在分布式锁高并发下的优化**


当某个冷门用户数据早已过期，但由于热搜等原因突然出现高并发读时，就会出现大量并发线程从缓存里都读不到数据，然后都会尝试进行读库 \+ 写缓存。也就是有大量并发线程在执行getUserInfoFromDB()方法，出现缓存击穿问题。


 


这些线程中只会有一个线程获取到锁，而其他并发的线程则产生严重的锁竞争问题。进行锁竞争的线程会串行化排队，第一个获取到锁的线程读库 \+ 写缓存。后续的线程获取到锁后，通过双重检查就可以直接读缓存了。但是即便有了双重检查，这些排队获取锁的线程还是需要一个个串行获取锁后才能执行。


 


然而其实只要第一个线程拿到锁，完成读库 \+ 写缓存后，Redis缓存里就已经有数据了。其他正在串行化排队获取锁的线程，就没必要继续排队去获取锁了。因此只要第一个线程成功完成读库 \+ 写缓存，其他线程就可以转为无锁情况下的串行读缓存。


 


为此，可以设置分布式锁的超时时间，超时时间可以参考一个线程完成读库 \+ 写缓存的时间。这样在超时时间内没有获得锁的线程会等待，超过超时时间内还获取不到锁就会返回false。当这些排队的线程获取分布式锁超时而返回false后，就可以尝试转为无锁串行读缓存了。


 


**9\.利用分布式锁自动超时消除串行等待锁的影响**


进行加锁时设置锁的超时时间，让排队获取锁的线程自动超时。第一个拿到锁的线程在超时时间内处理完事情会释放锁，其他线程会继续竞争锁。而在这个超时时间里没有获得锁的线程会被挂起并进入队列进行串行等待。如果在这个超时时间外还获取不到锁，排队的线程就会被唤醒并返回false。而获取分布式锁超时失败的线程通过再次读缓存，从而实现无锁串行读缓存。


 


注意：设置的自动超时时间并不好控制。因此可以参考AQS的做法，获取不到锁的线程先挂起，第一个释放锁的线程就把这些线程全都唤醒执行并发读缓存。AQS是对排队中的线程一个个进行唤醒，需要改造成第一个锁释放后全部排队的线程都唤醒。



```
private CookbookUserDTO getUserInfoFromDB(Long userId) {
    //首先获取分布式锁，防止出现两个读写线程造成缓存和DB不一致，这个锁和更新用户数据时用来保证幂等性的锁是同一把锁
    String userLockKey = RedisKeyConstants.USER_UPDATE_LOCK_PREFIX + userId;
    boolean lock = false;
    try {
        //尝试加锁并且设置锁的超时时间
        //第一个拿到锁的线程在超时时间内处理完事情会释放锁，其他线程会继续竞争锁
        //而在这个超时时间里没有获得锁的线程会被挂起并进入队列进行串行等待
        //如果在这个超时时间外还获取不到锁，排队的线程就会被唤醒并返回false
        lock = redisLock.tryLock(userLockKey, USER_UPDATE_LOCK_TIMEOUT);
    } catch(InterruptedException e) {
        CookbookUserDTO user = getUserFromCache(userId);
        if (user != null) {
            return user;
        }
        log.error(e.getMessage(), e);
        throw new BaseBizException("查询失败");
    }
    //获取分布式锁超时失败
    if (!lock) {
        //并发进来串行排队的线程获取分布式锁超时返回失败后，就重新读缓存(实现"串行等待锁+读缓存"转"串行读缓存")
        CookbookUserDTO user = getUserFromCache(userId);
        if (user != null) {
            return user;
        }
        log.info("缓存数据为空，从数据库查询用户数据时获取锁失败，userId:{}", userId);
        throw new BaseBizException("查询失败");
    }
    ...
}
```

 


**10\.写少读多的企业级缓存架构设计总结**


**(1\)读多写少场景引入Redis缓存**


**(2\)同步双写实现数据库和缓存强一致性**


**(3\)读多写少场景下的数据库和缓存双写企业级方案**


 


**(1\)读多写少场景引入Redis缓存**


用户数据是典型读多写少的数据，可能0\.01%写，99\.99%读。读多写少的场景引入Redis缓存是非常有必要的，因为读可能是高并发的。而且读请求没必要都从数据库里读取数据，从缓存中读取数据即可。


 


**(2\)同步双写实现数据库和缓存强一致性**


关于如何写库和写缓存才能保证数据库和缓存一致性，有两个方案：


方案一：通过异步先写数据库然后根据binlog写缓存实现最终一致性


方案二：通过同步使用分布式锁双写数据库和缓存实现强一致性


 


这里采用了同步双写，因为比较简单。当进行写时，会将数据库和缓存一起写，并且设置过期时间，以便让缓存留下热数据。当进行读时，每次读取缓存都会自动expire延期，让热数据一直保留在缓存里，冷数据自动过期。当需要读取冷数据时，发现没从缓存中读取到，那么再去读库 \+ 写缓存。


 


通过过期时间 \-\> 实现冷热分离 \-\> 让冷数据停留在MySQL \+ 让热数据停留在Redis


 


**(3\)读多写少场景下的数据库和缓存双写企业级方案**


一.数据库和缓存同步双写


二.缓存实现冷热分离：设置过期时间 \+ 自动expireTime延期


三.缓存惊群解决方案：随机过期时间


四.缓存穿透解决方案：根据key查库发现不存在，可以缓存空数据


五.数据库缓存强一致性：写库和读库使用同一分布式锁 \+ 读库前进行双重检查


六.缓存击穿问题：分布式锁 \+ 消除串行等待锁(读操作的分布式锁设置自动超时)


 


 本博客参考[FlowerCloud机场](https://hushicha.org)。转载请注明出处！
