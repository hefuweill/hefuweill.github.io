---
title: Android 四大组件之 ContentProvider 篇
date: 2018-04-15 14:45:53
type: "Android"
---

## 前言

上篇文章总结了 BroadcastReceiver，这篇文章来复习下四大组件的最后一个 ContentProvider，其能将应用程序内部存储的数据通过其所提供的访问方式分享给其它应用程序使用，先来看看系统提供的 ContentProvider。<!-- more -->

## 系统提供的 ContentProvider

系统提供了各式各样的ContentProvider,比如通讯录、短信等等，这里以获取通讯录中联系人信息为例。

```kotlin
data class Contact(val id: Int, val name: String, val phone: String)

private fun getContacts(): List<Contact> {
    val cursor = contentResolver.query(ContactsContract.Contacts.CONTENT_URI, null,
            null, null, null)
    val contacts = ArrayList<Contact>()
    if (cursor != null) {
        while (cursor.moveToNext()) {
            val id = cursor.getInt(cursor.getColumnIndex(ContactsContract.Contacts._ID))
            val name = cursor.getString(cursor.getColumnIndex(ContactsContract.Contacts.DISPLAY_NAME))
            // 再根据ID查出电话
            val phoneCursor = contentResolver.query(ContactsContract.CommonDataKinds.Phone.CONTENT_URI, null,
                    ContactsContract.CommonDataKinds.Phone.CONTACT_ID + " = " + id, null, null)
            if (phoneCursor != null) {
                while (phoneCursor.moveToNext()) {
                    val phoneNumber = phoneCursor.getString(phoneCursor.getColumnIndex(
                            ContactsContract.CommonDataKinds.Phone.NUMBER))
                    contacts.add(Contact(id, name, phoneNumber))
                }
                phoneCursor.close()
            }
        }
        cursor.close()
    }
    return contacts
}
// 所需权限
<uses-permission android:name="android.permission.READ_CONTACTS" />
```

这里先通过 contentResolver.query 查询出 id 和 name ，然后再根据 id 去另一张表中查询 phoneNumber。

## 自定义 ContentProvider

需要创建一个 ContentProvider 的子类，重写以下几个方法

- ```onCreate``` 在应用程序启动时会调用（先于 Application.onCreate ），因为其运行在主线程所有不能执行耗时任务，不然可能会使程序启动过慢，或者直接 ANR。
- ```insert``` 在子线程运行，外界调用 ContentResolver.insert 时调用。
- ```query```  在子线程运行，外界调用 ContentResolver.query 时调用。
- ```update``` 在子线程运行，外界调用 ContentResolver.update 时调用。
- ```delete``` 在子线程运行，外界调用 ContentResolver.delete 时调用。
- ```getType``` 如果该 Ur i表示一条记录返回值应该以 vnd.android.cursor.item 开头，多条记录返回值应该以vnd.android.cursor.dir/ 开头。

然后再在清单文件中进行配置。

```xml
<provider
    android:authorities="com.hfw.provider"
    android:exported="true"
    android:name=".MyProvider"/>
```

这里的主机名确定了什么 Uri 能够调用该 ContentProvider，比如这里设置了 com.hfw.provider ，那么只有以 content://com.hfw.provider 开头的 Uri 才会调用该 ContentProvider ，主机名后面还可以跟上要操作的表明、或者某些条件（自己约定好就行）。

* 例如 content://com.hfw.provider/user 表示要操作 User 表中所有的数据、content://com.hfw.provider/user/zhangsan 表示要操作 User 表中 name 为 zhangsan 的记录。