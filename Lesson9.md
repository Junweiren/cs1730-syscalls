# Lesson 9 -  Building a Content Provider

In this lesson, you will be creating your own **Content Provider** class for your app. This note is based on this [Udacity Tutorial][1] made by Google.

### What is Content Provider?

> Content providers can help an application manage access to data stored by itself, stored by other apps, and provide a way to share data with other apps.
-- [Documentation][2]  

### Steps to Create a Provider

 1. Create a class that extends from the `ContentProvider` and implement the  `OnCreate()` function.
 2. Register your content provider in the **_AndroidManifest.xml_** file.
 3. Define URI's. To identify the Content and the different data types that it can return.
 4. Add these URI's to the **Contract** class. 
 5. Build a **URIMatcher** to match URI patterns to integers.
 6. Implement the required **CRUD** methods like _query_, *insert*, and _delete_.


**This note will be using ToDo List project as example from Udacity tutorial**
___
#### 1. Create Customized Content Provider
```java
public class Task ContentProvider extends ContentProvider{
    private TaskDbHelper mTaskDbHelper;
    
    @Override
    public boolean onCreate(){
        Context context = getContent();
        mTaskDbhelper = new TaskDbHelper(context);
    }
}
```
---
#### 2. Register Your Provider
You need to regiser your Content Provider in the **Android Manifest**, similar to how you declare any activitiy, so that it can be seen by the system and your app will be able to refer to it later on. And pass calls to it through a **Content Resolver** in your UI code.

In **Android Manifest**, you need to define authority, it's typically just the package name of the app that a provider belongs to. That way **ContentResolver** can make direct calls to **ContentProvider**.

Here is the code you need to add in **AndroidManifests.xml**.

```xml
<provider
            android:authorities="com.example.android.todolist"
            android:name="com.example.android.todolist.data.TaskContentProvider"
            android:exported="false"
 />
```
* **authorities**
    :   The full package name.
* **name**
    :   The full package name and class.
* **exported:** 
    :   It determines whether or not **ContentProvider** can be accessed by other applications. 

---
#### 3. Define URI's
**URIs**
1. Identify your provider
2. Identify different type of data that the provider can work with
3. Fill out URIMatcher

**Example:**
##### <scheme>//<authority>/<path>
`content://com.android.example.todolist/tasks`

**URIMatcher**
* "path" - matches "path" exactly
* "path/#" - matches "path" followed by a number
* "path/*" - matches "path" followed by a string
* "path/*/other/#" - matches "path" followed by a string followed by "other" followed by a number
------------------------------------------------------
 #### 4. Change the Contract
**URI breakdown:**

Component|Define as
---|---
Scheme | content://
Content authority | reference to the provider (com.example.android.todolist)
Base content | URI scheme + authority
Path | to specific data
Content URI|base content URI + path

 In **TaskContract** class add the following constants so the class looks like this:
  ```java
  package com.example.android.todolist.data;

import android.net.Uri;
import android.provider.BaseColumns;

public class TaskContract {
    public static final String AUTHORITY = "com.example.android.todolist";
    public static final Uri BASE_CONTENT_URI = Uri.parse("content://" + AUTHORITY);
    public static final String PATH_TASKS = "tasks";
  
 /* TaskEntry is an inner class that defines the contents of the task table */
    public static final class TaskEntry implements BaseColumns {
        public static final Uri CONTENT_URI = BASE_CONTENT_URI.buildUpon().appendPath(PATH_TASKS).build();
        public static final String TABLE_NAME = "tasks";
        public static final String COLUMN_DESCRIPTION = "description";
        public static final String COLUMN_PRIORITY = "priority";
       // Note: Because this implements BaseColumns, the _id column is generated automatically

    }
}
  ```
-------------------------------------------------------
#### 5. Build the URIMatcher
UriMatcher
:   Determines what kind of URI the provider receives and match it to an integer constant.

Add following code to **TaskContentProvider** class
```java
public class TaskContentProvider extends ContentProvider {

    //  Define final integer constants for the directory of tasks and a single item.
    // It's convention to use 100, 200, 300, etc for directories,
    // and related ints (101, 102, ..) for items in that directory.
    public static final int TASK = 100;
    public static final int TASK_WITH_ID = 101;

    public static UriMatcher sUriMatcher = buildUriMatcher();
    
    public static UriMatcher buildUriMatcher() {
        UriMatcher uriMatcher = new UriMatcher(UriMatcher.NO_MATCH);
        uriMatcher.addURI(TaskContract.AUTHORITY, TaskContract.PATH_TASKS, TASK);
        uriMatcher.addURI(TaskContract.AUTHORITY, TaskContract.PATH_TASKS + "/#", TASK_WITH_ID);
        return uriMatcher;
    }
    
...//more stuff
}
```
--------------------------------------------
**Little Note**
The flow from UI to Database:
UI --> Resolver(URI) --> Provider --> URIMatcher --> SQL Code --> Database 
Database then pass cursor back to UI
--------------------------------------------
#### Overview of Provider Functions
Profider Methods|Parameters|Return Type
---|---|---
onCreate|`(Context context)` |
insert|`(Uri uri, ContentValue value)` | Uri
query|`(Uri uri, String[] Projection, String selection, String[] selectionArgs, String sortOrder)` | Cursor
update|`(Uri uri, ContentValues value, String selection, String[] selectionArgs)` | int
delete|`(Uri uri, String selection, String[] selectionArgs)` | int
getType|`(Uri uri)`|String
-------------------------------
#### 6. Inplement CRUD method

### **Insert method** - in `TaskContentProvider.java`
```java
@Override
    public Uri insert(@NonNull Uri uri, ContentValues values) {
        final SQLiteDatabase sqLiteDatabase = mTaskDbHelper.getWritableDatabase();

        int matchingCode = sUriMatcher.match(uri);
        Uri returnUri;

        switch (matchingCode) {
            case TASKS:
                long id = sqLiteDatabase.insert(TABLE_NAME, null, values);
                if (id > 0) {
                    returnUri = ContentUris.withAppendedId(uri, id);
                } else {
                    throw new android.database.SQLException("Failed to insert row into " + uri);
                }
                break;
            default:
                throw new UnsupportedOperationException("Unknown Uri: " + uri);
        }
        getContext().getContentResolver().notifyChange(uri, null);
        return returnUri;
    }
```
### **UI code** - in `AddTaskActivity.java`
```java
    public void onClickAddTask(View view) {

        EditText taskEditText = (EditText) findViewById(R.id.editTextTaskDescription);
        String des = taskEditText.getText().toString();
        if (des.length() == 0) {
            return;
        }
        ContentValues contentValues = new ContentValues();
        contentValues.put(TaskContract.TaskEntry.COLUMN_DESCRIPTION, des);
        contentValues.put(TaskContract.TaskEntry.COLUMN_PRIORITY, mPriority);

        Uri retUri = getContentResolver().insert(TaskContract.TaskEntry.CONTENT_URI, contentValues);
        if (retUri != null) {
            Toast.makeText(this, retUri.toString(), Toast.LENGTH_LONG).show();
        }
        finish();
}
```
---------------------------------
### **Quert method** - in `TaskContentProvider.java`
```java
    @Override
    public Cursor query(@NonNull Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder) {

        final SQLiteDatabase db = mTaskDbHelper.getReadableDatabase();
        int matchingCode = sUriMatcher.match(uri);
        Cursor returnCursor;
        switch (matchingCode) {
            case TASKS:
                returnCursor = db.query(TABLE_NAME, projection, selection, selectionArgs, null,null, sortOrder);
                break;
            case TASK_WITH_ID:
                String id = uri.getPathSegments().get(1);
                String mSelection = "_id=?";
                String[] mSelectionArgs = new String[]{id};
                returnCursor = db.query(TABLE_NAME, projection, mSelection, mSelectionArgs, null, null, sortOrder);
                break;
            default:
                throw new UnsupportedOperationException("Unknown Uri: " + uri);

        }
        returnCursor.setNotificationUri(getContext().getContentResolver(), uri);
        return  returnCursor;
    }
```
### **UI code** - in AsyncTaskLoader of `MainActivity.java`
```java
@Override
    public Cursor loadInBackground() {
        try {
             return getContentResolver().query(TaskContract.TaskEntry.CONTENT_URI, null, null, null, TaskContract.TaskEntry.COLUMN_PRIORITY);
         } catch (Exception e) {
            e.printStackTrace();
            eturn null;
        }
     }
```
---------------------------------
### **Delete method** - in `TaskContentProvider.java`
```java
    @Override
    public int delete(@NonNull Uri uri, String selection, String[] selectionArgs) {

        final SQLiteDatabase db = mTaskDbHelper.getWritableDatabase();
        int matchCode = sUriMatcher.match(uri);
        int deleteId = 0;
        switch (matchCode) {
            case TASK_WITH_ID:
                String id = uri.getPathSegments().get(1);
                String mSelection = "_id=?";
                String[] mSelectionArgs = new String[]{id};
                deleteId = db.delete(TABLE_NAME, mSelection, mSelectionArgs);
                if (deleteId > 0) {
                    getContext().getContentResolver().notifyChange(uri , null);
                } else {
                    throw new android.database.SQLException("Failed to delete row  " + uri);
                }
                break;
            default:
                throw new UnsupportedOperationException("Unknown uri: " + uri);
        }
        return deleteId;
    }
```


  [1]:https://classroom.udacity.com/courses/ud851/lessons/88171055-d9e6-4da5-acc4-b92a302a75a8/concepts/a076b71c-620e-4dd8-900e-bb8b43f3314a
  [2]:https://developer.android.com/guide/topics/providers/content-providers.html
