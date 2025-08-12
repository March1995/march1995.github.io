---
title: Mybatis一级缓存问题
date: 2025-05-05
desc: 
keywords: mybatis
categories:
  - 日常问题记录
---

### 问题现象

```java 
// 根据packageId，获取包下面的规则和参数   ruleChoiceList数据库方法
List<RuleChoice> ruleChoiceList = packageRuleService.ruleChoiceList(packageId, ruleType, autoCheckBeforeCalculate);  
  
// 再加上飞行值勤期里的规则参数  
ruleChoiceList.forEach(ruleChoice -> {  
    ruleChoice.setPackageType(packageType);  
    // 飞值  
    if (FunctionHolder.clazzFunctionIdMap.get(FuncPairingPilotFlyDutyTime.class).contains(ruleChoice.getFunctionId()) )  
    ) {  
	    // 这里第二次进来的时候，第一次add的结果还在
        List<RuleParam> ruleParams = dutyTimeService.flyDutyConfigToRule(filialeCode, packageType);  
        ruleChoice.getParamList().addAll(ruleParams);  
    } 
});
```

### 分析过程&原因

- 参数个数不对，首先怀疑是代码写错了，后面打了日志，发现第一次是好的，第二次进来的时候个数不对。
- 怀疑第二次进来的时候 拿到的是第一次的请求结果，也就是第二次拿到的对象是第一次的返回对象，怀疑是mybatis的一级缓存原因。
- 特定条件才会出现，不是必须，怀疑是某个接口的问题。
- 排查哪个接口请求的时候，会调用2次这个方法，且还是是在同一个sqlsession中，在代码中就是在同一个事务中。因为不同的事务，sqlsession肯定不是同一个，不会出现一级缓存问题。
- 排查到有个接口在最外层加了一个大事务。其中在复现问题的过程中，发现内层的方法加上事务注解的时候，该方法不会在事务中，因为用的this调用，所以怀疑是在最外层加的事务。
- log.info("getRuleChoices 在事务中{}", TransactionSynchronizationManager.isActualTransactionActive()); 可以打印当前方法是否在事务中。

```java
// 如果在事务中，就不去用线程池去读了，会读不到未提交的  
if (TransactionSynchronizationManager.isActualTransactionActive()) {  
  
    SchPairingQuery schRosterQuery = new SchPairingQuery();  
    schRosterQuery.setPairingIds(pairingIdList);  
    List<SchPairingVO> pairingVOList = new ArrayList<>();  
    List<RosterFilialeSheetVO> sheetVOS = new ArrayList<>();  
    if (CollUtil.isNotEmpty(pairingIdList)) {  
		// 这里2个方法都去调用了 getPairings(schRosterQuery).getPairings() 里面回去重复计算规则告警 报错了
        pairingVOList = coreService.getPairings(schRosterQuery).getPairings();  
        sheetVOS = coreService.getPersonRosterSheet(schRosterQuery);  
    }  
    PersonGanttData ganttData = new PersonGanttData();  
    if (CollUtil.isNotEmpty(personIdList)) {  
        schRosterQuery = createQuery(pairingIdList, personIdList, startDate, endDate);  
        ganttData = coreService.getRosterGantt(schRosterQuery);  
    }  
  
    return new RosterOperResultVO(pairingVOList, sheetVOS, ganttData);  
}  
// 下面是通过线程池去调用，在不同的sqlseesion中，所以不会出问题
ThreadPoolExecutor executor = CrewReturnThreadPoolExecutor.getInstance();  
LoginUserInfo loginUserInfo = UserUtil.threadLocal.get();  
String moduleFlag = ModuleFlagHolder.threadLocal.get();  
Integer solutionId = SolutionIdHolder.threadLocal.get();  
Integer mirrorId = MirrorIdHolder.threadLocal.get();  
Future<PersonGanttData> personGanttData = executor.submit(() -> {  
    if (CollUtil.isEmpty(personIdList)) {  
        return new PersonGanttData();  
    }  
    SchPairingQuery schRosterQuery = createQuery(pairingIdList, personIdList, startDate, endDate);  
    ModuleFlagHolder.threadLocal.set(moduleFlag);  
    SolutionIdHolder.threadLocal.set(solutionId);  
    MirrorIdHolder.threadLocal.set(mirrorId);  
    UserUtil.threadLocal.set(loginUserInfo);  
    RetainManualResultHolder.threadLocal.set(true);  
    return coreService.getRosterGantt(schRosterQuery);  
});

```
### 解决方式：

当updateAllFlightTimeCache方法中数据库为空的时候，线程A和线程B同时调用getFlightTimeFromCache方法，线程A和线程B都会判断SCH_FLIGHT_TIME_CACHE.estimatedSize() == 0，
线程A先执行updateAllFlightTimeCache方法，线程B执行getFlightTimeFromCache方法，线程B会判断SCH_FLIGHT_TIME_CACHE.estimatedSize() == 0，此时线程B会再次执行updateAllFlightTimeCache方法，导致线程B会重复执行updateAllFlightTimeCache方法，最终导致缓存中数据重复。

### 解决办法：

1.在xml方法中 添加 flushCache="true"，不使用一级缓存
```java
<select id="ruleChoiceList" resultMap="choiceMap" flushCache="true">
```
2.事务粒度的问题，尽量不要使用大事务
