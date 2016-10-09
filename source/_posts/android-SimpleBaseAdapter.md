title: "Android简便通用SimpleBaseAdapter"
date: 2015-11-24 23:28:35
categories: android 
tags: [android, adapter]
toc: true
description: 在Android开发中ListView是必不可少的, 每个ListView都对应着一个Adapter。每次继承BaseAdapter的时候必须重写getCount(), getItem(), getItemId(), getView 这几个方法。而且为了优化列表使用的ViewHoder模式, 造成了大量的代码复写。所以可以抽象一个SimpleBaseAdapter减少代码复写。

---
SimpleBaseAdapter主要是解决了每次都要重写BaseAdapter那几个方法，还有优化了ViewHolder。我们可以先看看常见的Adapter写法。

###0x01 常见Adapter

常见的Adapter大家都很熟了，直接贴代码：

```java

public class NormalAdapter extends BaseAdapter {
    private Context context;
    private List<String> data;

    public NormalAdapter(Context context, List<String> data){
        this.context = context;
        this.data = data;
    }

    @Override
    public int getCount() {
        return data.size();
    }

    @Override
    public Object getItem(int position) {
        return data.get(position);
    }

    @Override
    public long getItemId(int position) {
        return position;
    }

    @Override
    public View getView(int position, View convertView, ViewGroup parent) {
        ViewHolder holder;

        if(convertView == null){
            holder = new ViewHolder();

            convertView = View.inflate(context, R.layout.menu_item, null);
            holder.tvContent = (TextView) convertView.findViewById(R.id.tv_content);
            convertView.setTag(holder);
        }else{
            holder = (ViewHolder) convertView.getTag();
        }
        holder.tvContent.setText(data.get(position));

        return convertView;
    }

    class ViewHolder{
        TextView tvContent;
    }
}

```

###0x01 抽象方法
我们可以直接抽象一个SimpleBaseAdapter。解决上面代码中每次都需要重写复写的方法。   
代码如下：

```java
public abstract class SimpleBaseAdapter<T> extends BaseAdapter {
    private Context mContext;
    private List<T> mData;

    public SimpleBaseAdapter(Context context, List<T> data) {
        this.mContext = context;
        this.mData = data == null ? new ArrayList<T>() : data;
    }

    @Override
    public int getCount() {
        return mData.size();
    }

    @Override
    public Object getItem(int position) {
        if (position >= mData.size())
            return null;
        return mData.get(position);
    }

    @Override
    public long getItemId(int position) {
        return position;
    }
}

```

###0x02 优化ViewHolder
上面代码只解决了代码复写的问题并没有getView()和ViewHolder，ViewHolder其实只要是作为ListView缓存的子选项，常见的写法是把子选项硬编码写在ViewHolder所以没有办法通用，我们可以考虑用列表保存选择子选项的控件。   
优化后的ViewHolder:

```java 
public class ViewHolder {
    private SparseArray<View> views = new SparseArray<>();
    private View convertView;

    public ViewHolder(View view) {
        this.convertView = view;
    }

    public <T extends View> T getView(int resId) {
        View view = views.get(resId);
        if (view == null) {
            view = convertView.findViewById(resId);
            views.put(resId, view);
        }
        return (T) view;
    }
}
```

SparseArray在代码理解上等价于HashMap<Integer, View>, SparseArray是Android提供的一个数据结构，但是比Map的效率更高。   

**<font color=red>PS: </font>** 如果你用到了`ExpandableListView` 需要重写BaseExpandableListAdapter那么上述的代码就不直接用了，但是ViewHolder还是可以通用的。所以如果项目中不止重写了BaseAdapter可以将ViewHolder提出来提高复用。

###0x03 源码
####完整SimpleBaseAdapter：

```java
public abstract class SimpleBaseAdapter<T> extends BaseAdapter {
    private Context mContext;
    private List<T> mData;

    public SimpleBaseAdapter(Context context, List<T> data) {
        this.mContext = context;
        this.mData = data == null ? new ArrayList<T>() : data;
    }

    @Override
    public int getCount() {
        return mData.size();
    }

    @Override
    public Object getItem(int position) {
        if (position >= mData.size())
            return null;
        return mData.get(position);
    }

    @Override
    public long getItemId(int position) {
        return position;
    }

    @Override
    public View getView(int position, View convertView, ViewGroup parent) {
        ViewHolder viewHolder;

        if(null == convertView){
            convertView = View.inflate(mContext, getItemResource(), null);
            viewHolder = new ViewHolder(convertView);
            convertView.setTag(viewHolder);
        }else{
            viewHolder = (ViewHolder) convertView.getTag();
        }
        return getItemView(position, convertView, viewHolder);
    }

    /**
     * 该方法需要子类实现，需要返回item布局的resource id
     *
     * @return resource id
     */
    public abstract int getItemResource();

    /**
     * 使用该getItemView方法替换原来的getView方法，需要子类实现
     *
     * @param position
     * @param convertView
     * @param viewHolder
     * @return
     */
    public abstract View getItemView(int position, View convertView, ViewHolder viewHolder);


    //数据操作
    public void addAll(List<T> elem) {
        mData.addAll(elem);
        notifyDataSetChanged();
    }

    public void remove(T elem) {
        mData.remove(elem);
        notifyDataSetChanged();
    }

    public void remove(int index) {
        mData.remove(index);
        notifyDataSetChanged();
    }

    public void replaceAll(List<T> elem) {
        mData.clear();
        mData.addAll(elem);
        notifyDataSetChanged();
    }
}

```

####使用示例
继承SimpleBaseAdapter，并且实现getItemResource和getItemView两个方法。   
示例:

```java
public class TestAdapter extends SimpleBaseAdapter<String> {

    public TestAdapter(Context context, List<String> data) {
        super(context, data);
    }

    @Override
    public int getItemResource() {
        return R.layout.menu_item;
    }

    @Override
    public View getItemView(int position, View convertView, ViewHolder viewHolder) {
        TextView tvContent = viewHolder.getView(R.id.tv_content);
        tvContent.setText((String)getItem(position));
        return convertView;
    }
}

```
