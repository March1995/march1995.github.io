---
title: Cache缓存双重检查锁未锁住
date: 2023-11-27 
desc:
keywords: cache 
categories: [日常问题记录]
---

### 背景

```java
 private static final Cache<String, SchFlightTime> SCH_FLIGHT_TIME_CACHE = Caffeine.newBuilder()
            .expireAfterWrite(1, TimeUnit.DAYS).build();

    public SchFlightTime getFlightTimeFromCache(String taskId, String taskTypeCode, String moduleFlag) {
        if (SCH_FLIGHT_TIME_CACHE.estimatedSize() == 0) {
            log.info("11");
            synchronized (SCH_FLIGHT_TIME_CACHE) {
                log.info("22");
                if (SCH_FLIGHT_TIME_CACHE.estimatedSize() == 0) {
                    log.info("33");
                    updateAllFlightTimeCache();
                }
            }
        }
        return SCH_FLIGHT_TIME_CACHE.getIfPresent(taskId + Constants.DASH + taskTypeCode + Constants.DASH + moduleFlag);
    }

    public void updateAllFlightTimeCache() {
        List<SchFlightTime> all = list();
        all.forEach(schFlightTime -> SCH_FLIGHT_TIME_CACHE.put(schFlightTime.getFlightId()
                + Constants.DASH + TaskTypeCodeEnum.FLIGHT.getCode() + Constants.DASH + schFlightTime.getModuleFlag(), schFlightTime));
    }

    public SchFlightTime getFlightTime(String taskId, String taskTypeCode, String moduleFlag) {
        return getOne(new LambdaQueryWrapper<SchFlightTime>()
                .eq(SchFlightTime::getFlightId, taskId)
//                .eq(SchFlightTime::getTaskTypeCode, taskTypeCode)
                .eq(StringUtils.isNotBlank(moduleFlag), SchFlightTime::getModuleFlag, moduleFlag));
    }
```

### 原因：

当updateAllFlightTimeCache方法中数据库为空的时候，线程A和线程B同时调用getFlightTimeFromCache方法，线程A和线程B都会判断SCH_FLIGHT_TIME_CACHE.estimatedSize() == 0，
线程A先执行updateAllFlightTimeCache方法，线程B执行getFlightTimeFromCache方法，线程B会判断SCH_FLIGHT_TIME_CACHE.estimatedSize() == 0，此时线程B会再次执行updateAllFlightTimeCache方法，导致线程B会重复执行updateAllFlightTimeCache方法，最终导致缓存中数据重复。

### 解决办法：

