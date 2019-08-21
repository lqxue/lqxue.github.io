---
title: RecyclerView条目包含EditText控制焦点获取、数据紊乱以及数据保存问题
date: 2019-08-16
tags:
	- View
	- RecyclerView
categories: Android
---


## 需求

RecyclerView条目中包含EditText,常规下只显示文本不叫其获的焦点类似TextView,当处于编辑状态时,要可以编辑,恢复成正常的EditText

## 问题

- 1.焦点问题；
- 2.EditText内数据紊乱问题；
- 3.EditText数据保存问题；

<!-- more -->

## 解决方案

```java
public class NewOrderAdapter extends RecyclerView.Adapter<NewOrderAdapter.ViewHolder> {

    private boolean canEdit;
    private List<ProductModel> dataList;

    public NewOrderAdapter(List<ProductModel> dataList) {
        this.dataList = dataList;
    }

    @Override
    public ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        View view = LayoutInflater.from(parent.getContext()).inflate(R.layout.items_order, parent, false);
        ViewHolder viewHolder = new ViewHolder(view);
        return viewHolder;
    }

    @Override
    public void onBindViewHolder(@NonNull RecyclerView.ViewHolder holder, final int position) {
        //点击了编辑状态
        if (canEdit) {
            tv_one_group_title.setFocusable(true);//设置输入框可聚集
            tv_one_group_title.setFocusableInTouchMode(true);//设置触摸聚焦
        } else {
            tv_one_group_title.setFocusable(false);//失去焦点
        }

        //给edittext设置默认值
        holder.requestNo.setText(dataList.get(position).getLnum());
        //设置标记
        holder.requestNo.setTag(position);
        holder.requestNo.addTextChangedListener(new TextWatcher() {
            @Override
            public void beforeTextChanged(CharSequence s, int start, int count, int after) {

            }

            @Override
            public void onTextChanged(CharSequence s, int start, int before, int count) {
                //关键点：1.给edittext设置tag，此tag用来与position做对比校验，验证当前选中的edittext是否为需要的控件;
                //2.焦点判断：只有当前有焦点的edittext才有更改数据的权限，否则会造成数据紊乱
                //3.edittext内数据变动直接直接更改datalist的数据值，以便滑动view时显示正确
                if ((Integer)holder.requestNo.getTag() == position && holder.requestNo.hasFocus()) {
                    dataList.get(position).setLnum(s.toString());
                }
            }

            @Override
            public void afterTextChanged(Editable s) {

            }
        });

    }


    @Override
    public int getItemCount() {
        return dataList.size();
    }

    public static class ViewHolder extends RecyclerView.ViewHolder {
        private TextView proNum;
        private TextView productName;
        private EditText requestNo;
        private TextView stock;
        private TextView transDay;
        private CheckBox contral;

        public ViewHolder(View itemView) {
            super(itemView);
            proNum = (TextView) itemView.findViewById(R.id.proNum);
            productName = (TextView) itemView.findViewById(R.id.productName);
            requestNo = (EditText) itemView.findViewById(R.id.requestNo);
            stock = (TextView) itemView.findViewById(R.id.stock);
            transDay = (TextView) itemView.findViewById(R.id.transDay);
            contral = (CheckBox) itemView.findViewById(R.id.contral);
        }
    }
}
```