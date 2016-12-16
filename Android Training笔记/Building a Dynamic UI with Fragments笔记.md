#Building a Dynamic UI with Fragments笔记
API小于11用FragmentActivity作为父容器。可以在xml文件直接添加<fragment/>标签。

>如果使用v7 appcompat library，应该使用AppCompatActivity替代FragmentActivity，它是FragmentActivity的子类。在xml里添加Fragment不能在运行期移除。

在运行期添加Fragment，activity layout中要有个容器(如空的FrameLayout)来插入Fragment。

当执行fragment transactions，例如替代或者移除，可以在提交FragmentTransaction之前调用addToBackStack()方法，使用户可以取消之前的替代或移除操作。

>当移除或替代一个fragment并且把transaction添加进后退栈，这个被移除(或替代)的fragment处于stopped状态(没有destroyed)。用户回退，这个fragment会restart。如果不把transaction添加进后退栈，这个fragment会被destroyed。

addToBackStack()的参数是为transaction指定一个唯一标识，如果不用到Fragment.BackStackEntry则可以不用指定。

Fragment之间的通讯是把Activity作为桥梁来实现，Fragment之间不应该直接进行通讯。

Activity和Fragment通讯步骤：

1.Fragment声明接口
2.Activity实现接口
3.Fragment调用Activity实现接口方法

步骤1和2

```java
public class HeadlinesFragment extends ListFragment {
    OnHeadlineSelectedListener mCallback;

    // Container Activity must implement this interface
    public interface OnHeadlineSelectedListener {
        public void onArticleSelected(int position);
    }

    @Override
    public void onAttach(Activity activity) {
        super.onAttach(activity);

        // This makes sure that the container activity has implemented
        // the callback interface. If not, it throws an exception
        try {
            mCallback = (OnHeadlineSelectedListener) activity;
        } catch (ClassCastException e) {
            throw new ClassCastException(activity.toString()
                    + " must implement OnHeadlineSelectedListener");
        }
    }

    ...
}
```

步骤3

```java
@Override
    public void onListItemClick(ListView l, View v, int position, long id) {
        // Send the event to the host activity
        mCallback.onArticleSelected(position);
    }
```

向一个Fragment发送消息

```java
public static class MainActivity extends Activity
        implements HeadlinesFragment.OnHeadlineSelectedListener{
    ...

    public void onArticleSelected(int position) {
        // The user selected the headline of an article from the HeadlinesFragment
        // Do something here to display that article

        ArticleFragment articleFrag = (ArticleFragment)
                getSupportFragmentManager().findFragmentById(R.id.article_fragment);

        if (articleFrag != null) {
            // If article frag is available, we're in two-pane layout...

            // Call a method in the ArticleFragment to update its content
            articleFrag.updateArticleView(position);
        } else {
            // Otherwise, we're in the one-pane layout and must swap frags...

            // Create fragment and give it an argument for the selected article
            ArticleFragment newFragment = new ArticleFragment();
            Bundle args = new Bundle();
            args.putInt(ArticleFragment.ARG_POSITION, position);
            newFragment.setArguments(args);

            FragmentTransaction transaction = getSupportFragmentManager().beginTransaction();

            // Replace whatever is in the fragment_container view with this fragment,
            // and add the transaction to the back stack so the user can navigate back
            transaction.replace(R.id.fragment_container, newFragment);
            transaction.addToBackStack(null);

            // Commit the transaction
            transaction.commit();
        }
    }
}
```








