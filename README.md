# 基于NotePad应用做功能扩展
## 实验要求
### 基本要求：
+  NoteList中显示条目增加时间戳显示
+  添加笔记查询功能（根据标题查询）
### 附加功能
+ UI美化(设置背景颜色，选中时的笔记背景颜色变换)
+ 更换背景
+ 修改字体大小(编辑时)
+ 笔记排序
+ 导出笔记
## 功能解析
+ 显示条目增加时间戳显示

    NotePad源码中的应用，在笔记列表中只显示了笔记标题。本功能对此进行扩展，在标题下方显示一个笔记创建/修改的时间。 

    1. 在noteslist_item.xml下增加一个TextView，这个TextView用于显示时间。
        ```xml
            <!--新增显示时间的TextView-->
        <TextView
                android:id="@+id/text1_time"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:textAppearance="?android:attr/textAppearanceSmall"
            android:textColor="#ffffff"
                android:paddingLeft="5dip"/>
        ```

    1. 在NotePadProvider.java中可以发现数据库结构定义时以有时间这个消息存储，现在需要的是把时间拿出来装填到列表中。
    
        所以在NoteList.java下对PROJECTION进行增加。
        ```java
        private static final String[] PROJECTION = new String[] {
                NotePad.Notes._ID, // 0
                NotePad.Notes.COLUMN_NAME_TITLE, // 1
                //扩展 显示时间戳
                NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE,
        };
        ```
        在dataColumns，viewIDs中补充时间戳：

        ```java
        String[] dataColumns = { NotePad.Notes.COLUMN_NAME_TITLE ,  NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE } ;
         ```
    1. 但完成以上后发现存入的时间是时间戳，所以需要将时间戳转换为时间。

        在NotePadProvider中的insert方法和NoteEditor中的updateNote方法中进行增加如下代码。一开始发现时间与北京时间差了八个小时，所以在时间上增加八个小时
        ```java
        //将时间戳转换为时间
        Long now = Long.valueOf(System.currentTimeMillis());
        //系统设置的是GMT-8,时间是从服务器上面传下来的,所以要加八小时
        now=now+8*60*60*1000;
        Date date = new Date(now);
        SimpleDateFormat format = new SimpleDateFormat("yyyy.MM.dd HH:mm:ss");
        String dateTime = format.format(date);
        values.put(NotePad.Notes.COLUMN_NAME_CREATE_DATE, dateTime);
        ```
    1. 模拟器截图：

        ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190404151708386.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjQ3OTEzNA==,size_16,color_FFFFFF,t_70)

        显示时间以为最后修改时间为准

+ 笔记查询功能（根据标题查询）

    1. 如果添加笔记查询功能，那么要在应用中添加一个搜索的按钮。先在list_options_menu.xml，添加一个搜索的按钮
        ```xml
            <!--搜索按钮-->
            <item
                android:id="@+id/menu_search"
                android:title="menu_search"
                android:icon="@android:drawable/ic_search_category_default"
                android:showAsAction="always">
        </menu>
        ```
    1. 在NoteList.java中d的onOptionsItemSelected方法里，往switch语句中添加搜索按钮的语句:
        ```java
        case R.id.menu_search:
                startActivity(new Intent(Intent.ACTION_SEARCH,getIntent().getData()));
                return true;
        ```
    1.  新建一个NoteSearch.java用于case语句中要跳转的搜索的activity。参考网上的信息后我发现搜索出来的也是笔记列表，所以这个activity可以模仿NoteList的activity继承ListActivity。在安卓中有个用于搜索控件叫SearchView，可以把SearchView跟ListView相结合，动态地显示搜索结果。
        - 新建布局文件note_search_list.xml（搜索页面布局）：
            ```xml
                <SearchView
                    android:id="@+id/search_view"
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:iconifiedByDefault="false"
                    android:queryHint="输入搜索内容..."
                    android:layout_alignParentTop="true">
                </SearchView>
                <ListView
                    android:id="@android:id/list"
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content">
                </ListView>
            ```
        - NoteSearch.java
            ```java
            public class NoteSearch extends ListActivity implements SearchView.OnQueryTextListener  {
                private static final String[] PROJECTION = new String[]{
                        NotePad.Notes._ID, 
                        NotePad.Notes.COLUMN_NAME_TITLE,
                        NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE,//显示修改时间的
                };
                @Override
                protected void onCreate(Bundle savedInstanceState) {
                    super.onCreate(savedInstanceState);
                    setContentView(R.layout.note_search_list);
                    Intent intent = getIntent();
                    if (intent.getData() == null) {
                        intent.setData(NotePad.Notes.CONTENT_URI);
                    }
                    SearchView searchview = (SearchView)findViewById(R.id.search_view);
                    //为查询文本框注册监听器
                    searchview.setOnQueryTextListener(NoteSearch.this);
                }
                @Override
                public boolean onQueryTextSubmit(String query) {
                    return false;
                }
                @Override
                public boolean onQueryTextChange(String newText) {
                    SearchView searchview = (SearchView)findViewById(R.id.search_view);
                    searchview.setOnQueryTextListener(NoteSearch.this);
                    String selection = NotePad.Notes.COLUMN_NAME_TITLE + " Like ? ";
                    String[] selectionArgs = { "%"+newText+"%" };
                    Cursor cursor = managedQuery(
                            getIntent().getData(),    
                            PROJECTION,               
                            selection,         
                            selectionArgs,     
                            NotePad.Notes.DEFAULT_SORT_ORDER 
                    );
                    String[] dataColumns = { NotePad.Notes.COLUMN_NAME_TITLE ,  NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE };
                    int[] viewIDs = { android.R.id.text1 , R.id.text1_time };
                    SimpleCursorAdapter adapter = new SimpleCursorAdapter(
                            this,                
                            R.layout.noteslist_item, 
                            cursor,
                            dataColumns,
                            viewIDs
                    );
                    setListAdapter(adapter);
                    return true;
                }
                @Override
                protected void onListItemClick(ListView l, View v, int position, long id) {
                    Uri uri = ContentUris.withAppendedId(getIntent().getData(), id);
                    String action = getIntent().getAction();
                    if (Intent.ACTION_PICK.equals(action) || Intent.ACTION_GET_CONTENT.equals(action)) {
                        setResult(RESULT_OK, new Intent().setData(uri));
                    } else {
                        startActivity(new Intent(Intent.ACTION_EDIT, uri));
                    }
                }
            }
            ```
        - 在AndroidManifest.xml注册notesearch的activity
            ```xml
            <activity
                android:name=".NoteSearch"
                android:label="NoteSearch"
                >
                <intent-filter>
                    <action android:name="android.intent.action.NoteSearch" />
                    <action android:name="android.intent.action.SEARCH" />
                    <action android:name="android.intent.action.SEARCH_LONG_PRESS" />
                    <category android:name="android.intent.category.DEFAULT" />
                    <data android:mimeType="vnd.android.cursor.dir/vnd.google.note" />
                </intent-filter>
            </activity>
            ```
    1. 模拟器截图

        ![SEARCH1](https://img-blog.csdnimg.cn/2019040916134669.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjQ3OTEzNA==,size_16,color_FFFFFF,t_70)

        ![SEARCH2](https://img-blog.csdnimg.cn/201904091614199.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjQ3OTEzNA==,size_16,color_FFFFFF,t_70)

+ UI美化(设置背景颜色，选中时的便签背景颜色变换)

    1. 在AndroidManifest.xml中增加一句代码将NotePad换为白色主题
        ```xml 
        <activity android:name="NotesList" android:label="@string/title_notes_list" android:theme="@android:style/Theme.Holo.Light">
        <activity
            android:name=".NoteSearch"
            android:label="NoteSearch"
            android:theme="@android:style/Theme.Holo.Light"
            >
        ```
    1. 让笔记的每条笔记都具有自己的背景颜色，所以选择在数据库中增加一个字段“color”用于存放每条笔记的背景颜色。先在NotePad.java 中设置颜色编号
        ```java
        public static final String COLUMN_NAME_BACK_COLOR = "color";
        public static final int DEFAULT_COLOR = 0;
        public static final int YELLOW_COLOR = 1;
        public static final int BLUE_COLOR = 2;
        public static final int GREEN_COLOR = 3;
        public static final int RED_COLOR = 4; 
         ```
    1. 在NotePadProvider.java中修改创建数据库的语句
    (如果数据库已创建需要将数据库删除重新运行一遍程序)
        ```java
        public void onCreate(SQLiteDatabase db) {
            db.execSQL("CREATE TABLE " + NotePad.Notes.TABLE_NAME + " ("
                    + NotePad.Notes._ID + " INTEGER PRIMARY KEY,"
                    + NotePad.Notes.COLUMN_NAME_TITLE + " TEXT,"
                    + NotePad.Notes.COLUMN_NAME_NOTE + " TEXT,"
                    + NotePad.Notes.COLUMN_NAME_CREATE_DATE + " INTEGER,"
                    + NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE + " INTEGER,"
                    + NotePad.Notes.COLUMN_NAME_BACK_COLOR + " INTEGER" //数据库增加color属性
                    + ");");
        }
        ```
    1. 实例化和设置静态对象的块
        ```java
        static{
            ·····
                    // add Maps "color" to "color"
            sNotesProjectionMap.put(
                NotePad.Notes.COLUMN_NAME_BACK_COLOR,
                NotePad.Notes.COLUMN_NAME_BACK_COLOR);
            ·····
        }
        ```
    1. 增加创建新笔记时需要执行的语句，给每条笔记设置默认背景颜色
        ```java
        // 笔记背景默认为白色
        if (values.containsKey(NotePad.Notes.COLUMN_NAME_BACK_COLOR) == false) {
            values.put(NotePad.Notes.COLUMN_NAME_BACK_COLOR, NotePad.Notes.DEFAULT_COLOR);
        }
        ```
    1. 笔记列表显示时从数据库中读取color的代码，所以在NoteList和NoteSearch中的PROJECTION添加color属性
        ```java
        private static final String[] PROJECTION = new String[] {
                NotePad.Notes._ID, // 0
                NotePad.Notes.COLUMN_NAME_TITLE, // 1
                //扩展 显示时间戳
                NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE,
                //扩展 显示笔记背景颜色
                NotePad.Notes.COLUMN_NAME_BACK_COLOR,
        };
        ```
    1. 使用bindView将颜色填充到ListView。新建一个MyCursorAdapter继承SimpleCursorAdapter
        ```java
        public class MyCursorAdapter extends SimpleCursorAdapter {
            public MyCursorAdapter(Context context, int layout, Cursor c,
                                String[] from, int[] to) {
                super(context, layout, c, from, to);
            }
            @Override
            public void bindView(View view, Context context, Cursor cursor){
                super.bindView(view, context, cursor);
                //从数据库中读取的先前存入的笔记背景颜色的编码，再设置笔记的背景颜色
                int x = cursor.getInt(cursor.getColumnIndex(NotePad.Notes.COLUMN_NAME_BACK_COLOR));
                switch (x){
                    case NotePad.Notes.DEFAULT_COLOR:
                        view.setBackgroundColor(Color.rgb(255, 255, 255));
                        break;
                    case NotePad.Notes.YELLOW_COLOR:
                        view.setBackgroundColor(Color.rgb(247, 216, 133));
                        break;
                    case NotePad.Notes.BLUE_COLOR:
                        view.setBackgroundColor(Color.rgb(165, 202, 237));
                        break;
                    case NotePad.Notes.GREEN_COLOR:
                        view.setBackgroundColor(Color.rgb(161, 214, 174));
                        break;
                    case NotePad.Notes.RED_COLOR:
                        view.setBackgroundColor(Color.rgb(244, 149, 133));
                        break;
                    default:
                        view.setBackgroundColor(Color.rgb(255, 255, 255));
                        break;
                }
            }
        }
        ```
    1. 将NoteList和NoteSearch里的适配器改为MyCursorAdapter
        ```java
        MyCursorAdapter adapter
            = new MyCursorAdapter(
                      this,
                      R.layout.noteslist_item,
                      cursor, 
                      dataColumns,
                      viewIDs
              );
        ```
    1. 新建colors.xml和color_select.xml。选中时的笔记时笔记的背景颜色会改变
        ```xml
        <resources>
            <color name="color1">#be96df</color>
        </resources>
        ```
        ```xml
        <selector xmlns:android="http://schemas.android.com/apk/res/android">
        <item android:drawable="@color/color1"
            android:state_pressed="true"/>
        </selector>
        ```
    1. 在notelist_item.xml里为控件添加选择器
        ```xml
        android:background="@drawable/color_select"
        ```
    1. 模拟器截图

        ![UI](https://img-blog.csdnimg.cn/20190411162203636.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjQ3OTEzNA==,size_16,color_FFFFFF,t_70)

        ![Ui](https://img-blog.csdnimg.cn/2019041117124366.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjQ3OTEzNA==,size_16,color_FFFFFF,t_70)

        ![ui](https://img-blog.csdnimg.cn/20190411162044452.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjQ3OTEzNA==,size_16,color_FFFFFF,t_70)

        ![select](https://img-blog.csdnimg.cn/20190412092631159.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjQ3OTEzNA==,size_16,color_FFFFFF,t_70)

+ 更换背景颜色

    1. 为了编辑笔记的时候可以显示背景颜色（显示时能查询到color数据），在NoteEditor.java中增加color属性
        ```java
        private static final String[] PROJECTION =
            new String[] {
                NotePad.Notes._ID,
                NotePad.Notes.COLUMN_NAME_TITLE,
                NotePad.Notes.COLUMN_NAME_NOTE,
                    //增加 背景颜色
                NotePad.Notes.COLUMN_NAME_BACK_COLOR
        };
        ```
    1. 在onResume()中增加设置背景颜色的代码
        ```java
        //读取颜色数据
        if(mCursor!=null){
            mCursor.moveToFirst();
            int x = mCursor.getColumnIndex(NotePad.Notes.COLUMN_NAME_BACK_COLOR);
            int y = mCursor.getInt(x);
            Log.i("NoteEditor", "color"+y);
            switch (y){
                case NotePad.Notes.DEFAULT_COLOR:
                    mText.setBackgroundColor(Color.rgb(255, 255, 255));
                    break;
                case NotePad.Notes.YELLOW_COLOR:
                    mText.setBackgroundColor(Color.rgb(247, 216, 133));
                    break;
                case NotePad.Notes.BLUE_COLOR:
                    mText.setBackgroundColor(Color.rgb(165, 202, 237));
                    break;
                case NotePad.Notes.GREEN_COLOR:
                    mText.setBackgroundColor(Color.rgb(161, 214, 174));
                    break;
                case NotePad.Notes.RED_COLOR:
                    mText.setBackgroundColor(Color.rgb(244, 149, 133));
                    break;
                default:
                    mText.setBackgroundColor(Color.rgb(255, 255, 255));
                    break;
            }
        }
        ```
    1. 在editor_options_menu.xml中添加一个更改背景的按钮
        ```xml
        <item android:id="@+id/menu_color"
        android:title="color"
        android:icon="@drawable/ic_menu_edit"
        android:showAsAction="always"/>
        ```
    1. 往NoteEditor.java中添加增加上一步新增的选项的执行语句，在onOptionsItemSelected()的switch中添加代码
        ```java
        case R.id.menu_color://跳转改变颜色的activity
            Intent intent = new Intent(null,mUri);
            intent.setClass(NoteEditor.this,NoteColor.class);
            NoteEditor.this.startActivity(intent);
            break;
        ```

    1. 新建选择背景颜色的布局note_color.xml
        ```xml
            <ImageButton
                android:id="@+id/color_white"
                android:layout_width="0dp"
                android:layout_height="50dp"
                android:layout_weight="1"
                android:background="#ffffff"
                android:onClick="white"/>
            <ImageButton
                android:id="@+id/color_yellow"
                android:layout_width="0dp"
                android:layout_height="50dp"
                android:layout_weight="1"
                android:background="#fdef90"
                android:onClick="yellow"/>
            <ImageButton
                android:id="@+id/color_blue"
                android:layout_width="0dp"
                android:layout_height="50dp"
                android:layout_weight="1"
                android:background="#6dc1ed"
                android:onClick="blue"/>
            <ImageButton
                android:id="@+id/color_green"
                android:layout_width="0dp"
                android:layout_height="50dp"
                android:layout_weight="1"
                android:background="#94e49f"
                android:onClick="green"/>
            <ImageButton
                android:id="@+id/color_red"
                android:layout_width="0dp"
                android:layout_height="50dp"
                android:layout_weight="1"
                android:background="#f19696"
                android:onClick="red"/>
        ```
    1. 新建选择颜色的NoteColor.java并注册
        ```java
       public class NoteColor extends Activity {
            private Cursor mCursor;
            private Uri mUri;
            private int color;
            private static final int COLUMN_INDEX_TITLE = 1;
            private static final String[] PROJECTION = new String[] {
                    NotePad.Notes._ID, 
                    NotePad.Notes.COLUMN_NAME_BACK_COLOR,
            };
            public void onCreate(Bundle savedInstanceState) {
                super.onCreate(savedInstanceState);
                setContentView(R.layout.note_color);
                mUri = getIntent().getData();
                mCursor = managedQuery(
                        mUri,       
                        PROJECTION,  
                        null,      
                        null,    
                        null        
                );
            }
            @Override
            protected void onResume(){
                if(mCursor.moveToFirst()){
                    color = mCursor.getInt(mCursor.getColumnIndex(NotePad.Notes.COLUMN_NAME_BACK_COLOR));
                    Log.i("NoteColor", "before"+color);
                }
                super.onResume();
            }
            @Override
            protected void onPause() {
                //将修改的颜色存入数据库
                super.onPause();
                ContentValues values = new ContentValues();
                Log.i("NoteColor", "cun"+color);
                values.put(NotePad.Notes.COLUMN_NAME_BACK_COLOR, color);
                getContentResolver().update(mUri, values, null, null);
                int x = mCursor.getColumnIndex(NotePad.Notes.COLUMN_NAME_BACK_COLOR);
                int y = mCursor.getInt(x);
                Log.i("NoteColor", "du"+y);
            }
            public void white(View view){
                color = NotePad.Notes.DEFAULT_COLOR;
                finish();
            }
            public void yellow(View view){
                color = NotePad.Notes.YELLOW_COLOR;
                finish();
            }
            public void blue(View view){
                color = NotePad.Notes.BLUE_COLOR;
                finish();
            }
            public void green(View view){
                color = NotePad.Notes.GREEN_COLOR;
                finish();
            }
            public void red(View view){
                color = NotePad.Notes.RED_COLOR;
                finish();
            }
        }
        ```
        ```java
        <activity android:name="NoteColor"
            android:theme="@android:style/Theme.Holo.Light.Dialog"
            android:label="Select Color"/>
        ```
    1. 模拟器截图

        ![change1](https://img-blog.csdnimg.cn/20190411164522350.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjQ3OTEzNA==,size_16,color_FFFFFF,t_70)

        ![change2](https://img-blog.csdnimg.cn/2019041116460551.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjQ3OTEzNA==,size_16,color_FFFFFF,t_70)

        ![change3](https://img-blog.csdnimg.cn/20190411164618517.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjQ3OTEzNA==,size_16,color_FFFFFF,t_70)

        ![change4](https://img-blog.csdnimg.cn/20190411171647862.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjQ3OTEzNA==,size_16,color_FFFFFF,t_70)

+ 修改字体大小(编辑时)

    1. 在editor_options_menu.xml中添加
        ```xml
        <item android:title="font size">
            <menu>
                <group>
                <item android:title="big"
                    android:id="@+id/font_40"></item>
                <item android:title="middle"
                        android:id="@+id/font_30"></item>
                <item android:title="small"
                        android:id="@+id/font_20"></item>
                </group>
            </menu>
        </item>
        ```
    1. NoteEditor的onOptionsItemSelected(MenuItem item)中添加相应的case来相应事件
        ```java
        case R.id.font_20:
            txt.setTextSize(20);
            break;
        case R.id.font_30:
            txt.setTextSize(30);
            break;
        case R.id.font_40:
            txt.setTextSize(40);
            break;
        ```
    1. 模拟器截图

        ![font1](https://img-blog.csdnimg.cn/20190412214506583.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjQ3OTEzNA==,size_16,color_FFFFFF,t_70)

        ![font2](https://img-blog.csdnimg.cn/20190412214529677.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjQ3OTEzNA==,size_16,color_FFFFFF,t_70)

        ![font3](https://img-blog.csdnimg.cn/20190412221349729.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjQ3OTEzNA==,size_16,color_FFFFFF,t_70)

+ 笔记排序

    1. 在list_options_menu.xml中添加
        ```xml
        <item android:title="sort">
                <menu>
                    <item
                        android:id="@+id/menu_sort1"
                        android:title="create time"/>
                    <item
                        android:id="@+id/menu_sort2"
                        android:title="update time"/>
                    <item
                        android:id="@+id/menu_sort3"
                        android:title="color"/>
                </menu>
        </item>
        ```
    1. NoteEditor的onOptionsItemSelected(MenuItem item)中添加相应的case来相应事件,并在外面添加cursor，adapter
        ```java
        private MyCursorAdapter adapter;
        private Cursor cursor;
        private String[] dataColumns = { NotePad.Notes.COLUMN_NAME_TITLE ,  NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE } ;
        private int[] viewIDs = { android.R.id.text1 , R.id.text1_time };
        ```
        ```java
        //创建时间排序
        case R.id.menu_sort1:
            cursor = managedQuery(
                    getIntent().getData(),
                    PROJECTION,
                    null,
                    null,
                    NotePad.Notes._ID
            );
            adapter = new MyCursorAdapter(
                    this,
                    R.layout.noteslist_item,
                    cursor,
                    dataColumns,
                    viewIDs
            );
            setListAdapter(adapter);
            return true;
        //修改时间排序
        case R.id.menu_sort2:
            cursor = managedQuery(
                    getIntent().getData(),
                    PROJECTION,
                    null,
                    null,
                    NotePad.Notes.DEFAULT_SORT_ORDER
            );
            adapter = new MyCursorAdapter(
                    this,
                    R.layout.noteslist_item,
                    cursor,
                    dataColumns,
                    viewIDs
            );
            setListAdapter(adapter);
            return true;
        //颜色排序
        case R.id.menu_sort3:
            cursor = managedQuery(
                    getIntent().getData(),
                    PROJECTION,
                    null,
                    null,
                    NotePad.Notes.COLUMN_NAME_BACK_COLOR
            );
            adapter = new MyCursorAdapter(
                    this,
                    R.layout.noteslist_item,
                    cursor,
                    dataColumns,
                    viewIDs
            );
            setListAdapter(adapter);
            return true;
        ```
    1. 模拟器截图（图片按创建时间-修改时间-颜色排序）

        ![sort1](https://img-blog.csdnimg.cn/20190421214432219.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjQ3OTEzNA==,size_16,color_FFFFFF,t_70)
        ![sort2](https://img-blog.csdnimg.cn/2019042121455766.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjQ3OTEzNA==,size_16,color_FFFFFF,t_70)
        ![sort3-create](https://img-blog.csdnimg.cn/20190421214617398.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjQ3OTEzNA==,size_16,color_FFFFFF,t_70)
        ![sort4-update](https://img-blog.csdnimg.cn/20190421214640678.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjQ3OTEzNA==,size_16,color_FFFFFF,t_70)
        ![sort5-color](https://img-blog.csdnimg.cn/20190421214735346.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjQ3OTEzNA==,size_16,color_FFFFFF,t_70)



+ 导出笔记
    1. 在editor_options_menu.xml中添加导出笔记按钮
        ```xml
        <item android:id="@+id/menu_output"
        android:title="output" />
        ```
    1. NoteEditor的onOptionsItemSelected(MenuItem item)中添加相应的case来相应事件
        ```java
        case R.id.menu_output://跳转导出笔记的activity
             Intent intent1 = new Intent(null,mUri);
             intent1.setClass(NoteEditor.this,Output.class);
             NoteEditor.this.startActivity(intent1);
             break;
        ```
    1. 新建布局output_text.xml
        ```xml
        <EditText android:id="@+id/name"
        android:maxLines="1"
        android:layout_width="wrap_content"
        android:ems="20"
        android:layout_height="wrap_content"/>
        <LinearLayout
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:orientation="horizontal">
        <Button
            android:id="@+id/output_ok"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:onClick="yes"
            android:text="yes" />
        <Button
            android:id="@+id/no"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:onClick="no"
            android:text="no" />
        </LinearLayout>
        ```
    1. 新建到导出笔记的Output.java并注册,添加权限
        ```java
        public class Output extends Activity {
            private static final String[] PROJECTION = new String[] {
                    NotePad.Notes._ID, 
                    NotePad.Notes.COLUMN_NAME_TITLE, 
                    NotePad.Notes.COLUMN_NAME_NOTE, 
                    NotePad.Notes.COLUMN_NAME_CREATE_DATE, 
                    NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE,
            };
            private Cursor mCursor;
            private Uri mUri;
            private String TITLE;
            private String NOTE;
            private String CREATE_DATE;
            private String MODIFICATION_DATE;
            //导出文件名
            private EditText mName;
            //标记 true导出,false不导出
            private boolean flag = false;
            private static final int COLUMN_INDEX_TITLE = 1;
            public void onCreate(Bundle savedInstanceState) {
                super.onCreate(savedInstanceState);
                setContentView(R.layout.output);
                mUri = getIntent().getData();
                mCursor = managedQuery(
                        mUri,     
                        PROJECTION,  
                        null,     
                        null,       
                        null         
                );
                mName = (EditText) findViewById(R.id.name);
            }
            @Override
            protected void onResume(){
                super.onResume();
                if (mCursor != null) {
                    mCursor.moveToFirst();
                    //默认的文件名为标题
                    mName.setText(mCursor.getString(COLUMN_INDEX_TITLE));
                }
            }
            @Override
            protected void onPause() {
                super.onPause();
                if (mCursor != null) {
                    TITLE = mCursor.getString(mCursor.getColumnIndex(NotePad.Notes.COLUMN_NAME_TITLE));
                    NOTE = mCursor.getString(mCursor.getColumnIndex(NotePad.Notes.COLUMN_NAME_NOTE));
                    CREATE_DATE = mCursor.getString(mCursor.getColumnIndex(NotePad.Notes.COLUMN_NAME_CREATE_DATE));
                    MODIFICATION_DATE = mCursor.getString(mCursor.getColumnIndex(NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE));
                    //true执行写文件
                    if (flag == true) {
                        write();
                    }
                    flag = false;
                }
            }
            public void yes(View v){
                flag = true;
                finish();
            }
            public void no(View v){
                finish();
            }
            private void write()
            {
                try
                {
                    if (Environment.getExternalStorageState().equals(
                            Environment.MEDIA_MOUNTED)) {
                        // 获取SD卡的目录
                        File sdCardDir = Environment.getExternalStorageDirectory();
                        //创建文件目录
                        File targetFile = new File(sdCardDir.getCanonicalPath() + "/" + mName.getText() + ".txt");
                        //写文件
                        PrintWriter ps = new PrintWriter(new OutputStreamWriter(new FileOutputStream(targetFile), "UTF-8"));
                        ps.println(TITLE);
                        ps.println(NOTE);
                        ps.println("创建时间：" + CREATE_DATE);
                        ps.println("修改时间：" + MODIFICATION_DATE);
                        ps.close();
                        Toast.makeText(this, "保存成功,保存位置：" + sdCardDir.getCanonicalPath() + "/" + mName.getText() + ".txt", Toast.LENGTH_LONG).show();
                    }
                }
                catch (Exception e)
                {
                    e.printStackTrace();
                }
            }
        }
        ```
        ```java
        <!-- 在SD卡中创建与删除文件权限 -->
        <uses-permission android:name="android.permission.MOUNT_UNMOUNT_FILESYSTEMS" />
        <!-- 向SD卡写入数据权限 -->
        <uses-permission 
        android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>
        ```
        ```java
        <activity android:name=".Output"
            android:label="please input name"
            android:theme="@android:style/Theme.Holo.Dialog">
        </activity>
        ```
    1. 模拟器截图

        ![output1](https://img-blog.csdnimg.cn/20190421214926790.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjQ3OTEzNA==,size_16,color_FFFFFF,t_70)
        ![output2](https://img-blog.csdnimg.cn/20190421214951166.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjQ3OTEzNA==,size_16,color_FFFFFF,t_70)
        ![output3](https://img-blog.csdnimg.cn/20190421215014739.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjQ3OTEzNA==,size_16,color_FFFFFF,t_70)
        ![output4](https://img-blog.csdnimg.cn/201904212150320.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjQ3OTEzNA==,size_16,color_FFFFFF,t_70)
        ![output5](https://img-blog.csdnimg.cn/20190421215049695.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjQ3OTEzNA==,size_16,color_FFFFFF,t_70)