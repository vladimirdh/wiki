---
title: 游览器跨tab通信
date: 2017-08-10
tags: 前端
---



最近整理了一下游览器同域跨tab通信的技术，写下来做个记录。



## LocalStorage

借助监听`storage`事件，可以很方便的达到效果。

```javascript
// tab A
 window.addEventListener('storage', (event) => {
   console.log(event.key, event.oldValue, event.newValue, event.url, event.storageArea)
 })

// tab B
localStorage.setItem('name', 'value')
```

优点：

- 给同一个`key`设置同样的`value`并不会触发事件：

  ```javascript
  localStorage.setItem('num', '0')
  localStorage.setItem('num', '1') //触发
  localStorage.setItem('num', '1') //不触发
  ```

- 使用事件驱动，API简单，获取到的`event`数据方便使用


缺点：

- 在不同的二级域名下无法触发

  ​

值得一提的是，根据[规范](https://www.w3.org/TR/webstorage/#the-storage-event)，`sessionStorage`也会触发`storage`事件，但实际测试下来并不能触发。因为`sessionStorage`在不同的tab间是独立的，所以只能在同一tab的不同`iframe`间触发



## Cookies

cookies的更新并没有相关事件，只能自己进行封装。

```javascript
// tab A
let cacheCookies = document.cookie
setInterval(() => {
	let newCookies = document.cookie
	if(newCookies === cacheCookies) return
	cacheCookies = newCookies
	console.log(newCookies)

}, 1000)

// tab B
document.cookie = 'name=value'
```

优点：

- 可以通过设置`domin`在不同的二级域名下触发：

  ```javascript
  document.cookie = 'name=newValue;domin=example.com;'
  ```



缺点：

- cookies大小有限制 //多大限制，超出后如何
- 同域的请求会自带cookies，造成了不必要的网络传输
- 每次获取到的是纯字符，需要自己判断是否更新和解析cookies

  ​



## IndexedDB & IDBObserver

`IndexedDB `是用来在客户端储存大量数据的，通过`IDBObserver`可以监听变化。`IndexedDB`的`API`比较复杂，有着强大的功能，跨tab通信只是附带，具体的功能不做讨论。

需要`Chrome`版本57以上并在`chrome://flags`打开`#enable-experimental-web-platform-features`。

```javascript
var dataNumber = 0;
var db = null;

var observer = new IDBObserver(changes => {
    var newData = changes.records.get('data')[0]
    console.log('changed data:', newData.type, newData.value)
    dataNumber = newData.value || 0
});

//初始化db
var openRequest = indexedDB.open('demoDB');

openRequest.onupgradeneeded = function () {
    openRequest.result.createObjectStore('data');
};

openRequest.onsuccess = function (event) {
    db = event.target.result;
    var transaction = db.transaction(['data'], 'readonly');
  	//监控db
    observer.observe(db, transaction, {
        operationTypes: ['put', 'clear'],
        values: true
    });

    var getRequest = transaction.objectStore('data').get('key');
    getRequest.onsuccess = (event) => {
        var result = event.target.result;
        dataNumber = result || 0
        console.log(`data number start from ${dataNumber}`)
    };
};
//清除data
function clearData() {
    if (!db) {
        console.log('Database connection still opening.');
        return;
    }
    var transaction = db.transaction(['data'], 'readwrite');
    var request = transaction.objectStore('data').clear();
    request.onsuccess = function () {
        transaction.oncomplete = function () {
            console.log('clear data completed')
        };
    };
}
//更新data
function updateData() {
    if (!db) {
        console.log('Database connection still opening.');
        return;
    }
    var newDataNumber = dataNumber + 1;
    var transaction = db.transaction(['data'], 'readwrite');
    var request = transaction.objectStore('data').put(newDataNumber, 'key');
    request.onsuccess = function () {
        transaction.oncomplete = function () {
            console.log('put data completed')
        };
    };
}

//tab A
updateData()

//tab B
clearData()
```



优点：

- 功能强大

缺点：

- 对于通信来说过于复杂



## 总结

本文简单罗列了纯客户端跨tab的技术，很显然，`LocalStorage`在多tab间通信有着明显的优势，只需要简单的封装就可使用。而`Cookies`虽然可以在不同的二级域名间通信，但也有一系列的缺点，实际使用中并不推荐。`IndexedDB`虽然功能强大，对其本意是储存数据而出现的，且对游览器的较高要求和复杂的`API`使其在跨tab通信这件事上并不是一个好的选择。





## 参考资料

- https://www.w3.org/TR/webstorage/#the-storage-event
- https://stackoverflow.com/questions/10260108/how-do-i-bind-event-to-sessionstorage#answer-21283114
- https://developer.mozilla.org/en-US/docs/Web/API/Document/cookie
- https://googlechrome.github.io/samples/indexeddb-observers/
- https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API/Using_IndexedDB