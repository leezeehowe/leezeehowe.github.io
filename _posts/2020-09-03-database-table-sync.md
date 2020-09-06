---
layout: post
title: "Typescript 数据库表数据同步"
subtitle: 'Typescript Interface'
author: "lee"
header-style: text
tags:
  - Typescript
  - Database
---

# 数据库表更新API

## 一、背景

POS为了实现功能离线可用，在用户电脑本地维护了一套数据库，由此产生了用户本地数据与后端数据不一致的问题，所以需要加入一个更新本地数据库的功能。

## 1. 1 表更新业务流程

抽象点理解就是

1. 从后端提供的API（一般是分页接口，视数据量决定）中获取到后端数据库的最新数据。
2. 以合适的方式（实现一致性、高性能）把最新数据更新到用户本地数据库。

但为了避免出现本地数据库与后端数据库数据不一致以及因为需更新的数据量过大导致用户电脑内存溢出或因频繁操纵数据库导致的性能问题，实际的表更新业务流程远没那么简单，具体流程为

1. 创建需更新数据的正式表的临时表。

2. 从后端API中请求最新数据，并先存储到内存中，当数据量到达一定程度再插入到临时表中。

   > 因为一般是分页接口，所以这里是一个递归过程。

3. 最新数据请求完毕后，检查临时表中的数据总量与后端API响应体中的数据总量字段是否相等，若不相等则数据不一致，此次数据更新失败。

   > 解决数据一致性的问题。

4. 把临时表中的数据插入到正式表中。

   > 这里有两种方式，全量或增量。全量会先把正式表truncate，增量则不会。

5. 删除临时表。

6. 结束。

## 二、实现

上诉流程中，对数据库中的任何一个表来说基本都是一样的，若有些流程存在差异也可通过配置的方式解决，而不必对每个表的数据更新都写一套代码。所以这实现了一个API通过接收配置的方式对不同的表进行数据更新。上诉业务在代码实现上是没什么难点的，重点在于对流程的抽象，体现在代码上即配置项的设计，所以该表更新API的实现重点和使用难点就在于配置项。

### 2.1 配置项

```javascript
/**
 * 单个数据同步任务的配置主体
 */
export interface SyncDataByTempTableOptions {
    /**
     * 需更新数据的表的所属数据库名
     */
    db: string,
    /**
     * 需更新数据的表的表名
     */
    table: string,
    /**
     * 从后端分页获取数据的API的配置
     */
    apiOptions: ApiOptions,
    /**
     * 内存容量
     */
    tempStoreCapacity: number,
    /**
     * 增量或全量更新，可接收一个工厂函数
     */
    mode: 'increment' | 'whole' | FunctionWithGlobalData<'increment' | 'whole'>,
    /**
     * 共享数据，可接收一个工厂函数，该共享数据作为所有工厂函数的入参。
     */
    globalData?: GlobalDataBuilder | GlobalData
}
    
/**
 * 关于数据同步中用于获取数据的RESTFUL API的配置 
 */
interface ApiOptions {
    /**
     * api函数
     * @param p 请求参数 
     */
    api(...p: any): Promise<any>,
    /**
     * api函数的请求参数，即@param api 的方法签名的形参，可接收一个工厂函数。
     */
    requestParams?: Array<any> | object | FunctionWithGlobalData<Array<any> | object>,
    /**
     * 要获取第几页的数据：
     * api函数的入参中，requestParams若为数组，则该字段指定数组下标。若是对象，指定键名。
     * @default 1
     */
    page?: string | number
    /**
     * api响应体的映射。该配置的原因是实现对api不同的响应体做的适配。
     * 因为一般是分页API，所以更新流程中需要获取到api的响应体中分页信息的字段，但是不同API的响应体的键名	 * 或许不同，所以暴露一个配置项指定键名。
     */
    responseParamsMap: {
        /**
         * 实际的业务数据
         */
        data: Array<string> | 'root',
        /**
         * 当前页数
         * @default 1
         */
        currentPage?: Array<string>,
        /**
         * 总页数，默认1
         * @default 1
         */
        totalPage?: Array<string>,
        /**
         * 总数量，默认是data的数量
         * @default 当前页的大小
         */
        totalRecord?: Array<string>
    },
    /**
     * 接收一个函数，可在把数据存储到内存前对其进行改变
     */
    transform?: Transform
}
    
/**
 * 在插入数据到临时表前对数据进行的一些自定义操作
 */
interface Transform {
    (dataFromApi: any): Array<any>
}
```

通过上面的配置，基本已经可以实现对大部分表的数据更新了，因为表数据更新过程中，还需要进行进度的展示，所以涉及一些对UI的操作，为了避免表更新代码中夹杂UI操作代码，所以用发布订阅模式对其进行解耦，在某些流程发布一个事件供UI操作模块进行相关操作。

```javascript
        /**
         * 表更新API: 开始一个任务前
         */
        SYNCDATA_ON_BEGIN: 'SYNCDATA_ON_BEGIN',
        /**
         * 表更新API: 一个任务成功后
         */
        SYNCDATA_ON_SUCCESS: 'SYNCDATA_ON_SUCCESS',
        /**
         * 表更新API: 单个任务失败
         */
        SYNCDATA_ON_ERROR: 'SYNCDATA_ON_ERROR',
        /**
         *  表更新API：请求API前
         */
        SYNCDATA_BEFORE_API: 'SYNCDATA_BEFORE_API',
        /**
         * 表更新API: 请求API后
         */
        SYNCDATA_AFTER_API: 'SYNCDATA_AFTER_API',
        /**
         *  表更新API: 更新表前
         */
        SYNCDATA_BEFORE_UPDATE_TABLE: 'SYNCDATA_BEFORE_UPDATE_TABLE',
        /**
         * 表更新API: 批量任务开始前
         */
        SYNCDATA_START: 'SYNCDATA_START',
        /**
         * 表更新API: 批量任务结束后
         */
        SYNCDATA__END: 'SYNCDATA_END'
```

### 2.2 数据更新业务流程的大体实现

```typescript
/**
 * 通过临时表的方式同步数据
 * @param options 
 */
const syncDataByTempTable = async function (options: SyncDataByTempTableOptions) {
    // 临时表
    let tempTb: any
    // 发布者
    const publisher = new Publisher(options.db, options.table, eventBus)
    publisher.publish(EventType.SYNCDATA_ON_BEGIN)
    try {
        // 创建临时表
        tempTb = await DaoCore.tempTableManagement.create(options.db, options.table)
        // 从后端API中获取数据并分批插入临时表中
        await fetchDataFromBackend(options.apiOptions)
        publisher.publish(EventType.SYNCDATA_BEFORE_UPDATE_TABLE)
        // 把临时表中的数据插入到正式表中
        switch (options.mode) {
            case 'increment': await doIncrementSync(options.db, options.table); break;
            case 'whole': await doWholeSync(options.db, options.table); break;
        }
    } catch (error) {
        // 同步失败
        throw error
    } finally {
        // 删除临时表，不抛出异常
        await DaoCore.tempTableManagement.drop(options.db, options.table, false)
    }
}
```

### 2.3 对外暴露的接口

对外暴露一个数据更新任务的执行者，通过接收用户指定的配置，对一个或多个表进行数据更新，途中通过发布事件的方式供UI操作模块进行流程展示相关的操作。

```typescript
export default class SyncDataTaskExecutor {

    options: Array<SyncDataByTempTableOptions>

    constructor(options: Array<SyncDataByTempTableOptions>) {
        this.options = options
    }

    async run() {
        const publisher = new Publisher('', '', eventBus)
        publisher.publish(EventType.SYNCDATA_START)
        for (const option of this.options) {
            publisher.db = option.db
            publisher.table = option.table
            try {
                // 这里先跑一遍配置项中的工厂函数
                await SyncDataOptionsFactory.build(option)
                // 开始数据更新流程
                await syncDataByTempTable(option)
                publisher.publish(EventType.SYNCDATA_ON_SUCCESS, { total: this.options.length })
            } catch (error) {
                error.message = `${option}`
                publisher.publish(EventType.SYNCDATA_ON_ERROR, {
                    error
                })
            }
        }
        publisher.publish(EventType.SYNCDATA__END)
    }
}
```

大体实现如上。

## 三、使用

### 3.1 编写配置

```javascript
/**
 * 表更新的配置
 */
const syncTableOptions: Array<SyncDataByTempTableOptions> = [
    {
        db: 'normal',
        table: 'goods',
        globalData: async () => {
            const statusTime = await GoodsDao.getLatestStatusTime()
            return {
                statusTime
            }
        },
        mode: function (globalData) {
            return globalData.statusTime ? 'increment' : 'whole'
        },
        tempStoreCapacity: 1000,
        apiOptions: {
            api: GoodsApi.listProductForOffline,
            page: 0,
            requestParams: (globalData) => {
                return [1, new Date(globalData.statusTime || '1999-1-1 00:00:00')]
            },
            responseParamsMap: {
                data: ['lists'],
                currentPage: ['current_page'],
                totalPage: ['total_page'],
                totalRecord: ['total_record']
            }
        }
    },
    {
        db: 'normal',
        table: 'productSettings',
        globalData: async () => {
            const statusTime = await ProductSettingsDao.getLatestStatusTime()
            return {
                statusTime
            }
        },
        mode: (globalData) => {
            return globalData.statusTime ? 'increment' : 'whole'
        },
        tempStoreCapacity: 1000,
        apiOptions: {
            api: ProductSettingsApi.listProductSettings,
            page: 0,
            requestParams: (globalData) => {
                return [1, new Date(globalData.statusTime || '1999-1-1 00:00:00')]
            },
            // 这里的键名配置项是数组是为了可以深搜索键名，如['data', 'name']代表data.name
            responseParamsMap: {
                data: ['list'],
                currentPage: ['current_page'],
                totalPage: ['total_page'],
                totalRecord: ['total_record']
            }
        }
    }
    // ......
]
```

上面的responseParamsMap键名配置项是数组是为了可以深搜索键名，如['data', 'name']代表data.name。

具体实现也很简单

```typescript
/**
 * 递归搜素一个对象中指定key的值，若无指定key值或指定
 * @param target 对象
 * @param keys 路径
 * @param times 不用传，传了会出错
 */
export const deepSearchValue = function <T>(target: object, keys: Array<string>, times: number = 0): T | undefined {
    if (typeof target !== 'object') {
        return undefined
    }
    if (times + 1 >= keys.length) {
        return target[keys[times]]
    }
    return deepSearchValue(target[keys[times]], keys, times + 1)
}
```

### 3.2 实例化任务执行者

```typescript
// 传入配置
const syncDataTaskExecutor = new SyncDataTaskExecutor(syncTableOptions)
// 监听任务异常
syncDataTaskExecutor.on(eventBusKey.SYNCDATA_ON_ERROR, (db: string, table: string, data: any) => {
    // 目前对于数据更新失败仅做日志记录
	console.log(`${db}/${table}表同步数据异常`, data.error)
})
// 执行数据更新任务
await syncDataTaskExecutor.run();
```

## 三、写到最后

通过上诉配置的方式，实现一个表的数据更新最快可能只需10秒...

```typescript
    {
        db: 'normal',
        table: 'goods_category',
        // 全量更新
        mode: 'whole',
        // 每当内存中存储满100条记录，就把该100条记录插入到临时表
        tempStoreCapacity: 100,
        apiOptions: {
            api: GoodsCategoryApi.getProductCategoryList,
            // 代表响应体中的object字段就是业务数据，即非分页API。
            responseParamsMap: {
                data: 'root'
            }
        }
    }
```

因为该表更新API尚且不能脱离于POS项目单独存在，还是与POS的`dao`层紧密耦合的，这也是称其为API的原因。

可优化的地方：

1. 可把API的响应体数据也纳入到配置项中的`globalData`中。
2. 配置项中的`tempStoreCapacity`也可接收一个工厂函数，通过API的响应体中的`total_record`即分页总量来动态的决定内存容量。例如可规定当内存中存储了此次待更新数据总量的1/3时就把内存中存储的数据插入到临时表中。
3. 任务执行者`SyncDataTaskExecutor`应该全局单例，避免出现同时存在两个任务执行者对同一个表进行数据更新的情况。

最后，typescript真香。