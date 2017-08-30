# Intent.FLAG_ACTIVITY_FORWARD_RESULT作用
##适用情况
多个Activity的值传递。ActivityA到达ActivityB再到达ActivityC，但ActivityB为过渡页可以finish了，此时ActivityC将值透传至ActivityA。

##代码实现
第一个页面：A跳到B

```java
public class AActivity extends Activity{
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Button button=new Button(this);
        button.setText("B");
        button.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Intent intent=new Intent(v.getContext(),BActivity.class);
                startActivityForResult(intent,1);
            }
        });
        setContentView(button);
    }

    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        System.out.println("A:" + (data == null ? data : data.getStringExtra("data")));
    }
}
```

过渡页：中间有再多过渡页也是一样。B跳到C

```java
public class BActivity extends Activity {
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Button button = new Button(this);
        button.setText("C");
        button.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Intent intent = new Intent(v.getContext(), CActivity.class);
                intent.setFlags(Intent.FLAG_ACTIVITY_FORWARD_RESULT);
                startActivity(intent);
                finish();
            }
        });
        setContentView(button);
    }

    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        System.out.println("B:" + (data == null ? data : data.getStringExtra("data")));
    }
}
```

C传值然后在A的OnActivityResult中获取值

```java
public class CActivity extends Activity{
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Button button=new Button(this);
        button.setText("result");
        button.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Intent intent=new Intent();
                intent.putExtra("data","data");
                setResult(Activity.RESULT_OK,intent);
                finish();
            }
        });
        setContentView(button);
    }
}
```

输出结果：
System.out: A:data

>注意：中间页面不能调用startActivityForResult方法，否则回报android.util.AndroidRuntimeException: FORWARD_RESULT_FLAG used while also requesting a result。中间页面需要finish后前面页面才能收到返回结果。


