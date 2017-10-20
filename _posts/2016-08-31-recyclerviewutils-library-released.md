---
layout: post
author: adam
modified: 2016-08-31
title: RecyclerViewUtils Library Released
published: true
tags: [RecyclerView, library]
description: RecyclerViewUtils is a library that removes some of the RecyclerView boilerplate code.
categories: Android
---

Recently, I became tired of writing the same old tedious code for every single RecyclerView and Adpater class I used, that all did the same thing, so I extrapolated all of it into a personal library.

The [RecyclerViewUtils](https://github.com/adammc331/RecyclerViewUtils) library helps make everyone's life a little easier with a CoreViewHolder and CoreAdapter class, described below.

<!--more-->

# CoreViewHolder

At the heart of these classes is [CoreViewHolder](https://github.com/AdamMc331/RecyclerViewUtils/blob/master/lib/src/main/java/com/adammcneilly/recyclerviewutils/CoreViewHolder.java) which is a RecyclerView.ViewHolder class used to display an object of a specific type. It has one abstract method for binding an object of that type.

Here is a sample of a CoreviewHolder for Account objects:

```java
   public class AccountViewHolder extends CoreViewHolder<Account> {
      private TextView tvName;
      private TextView tvBalance;
    
      public AccountViewHolder(View view) {
         super(view);
    
         this.tvName = (TextView) view.findViewById(R.id.account_name);
         this.tvBalance = (TextView) view.findViewById(R.id.account_balance);
      }
    
      @Override
         protected void bindItem(Account item) {
         this.tvName.setText(item.getName());
         this.tvBalance.setText(String.valueOf(item.getBalance()));
      }
   }
```

# CoreRecyclerViewAdapter

Using the CoreViewHolder class from above, the [CoreRecyclerViewAdapter](https://github.com/AdamMc331/RecyclerViewUtils/blob/master/lib/src/main/java/com/adammcneilly/recyclerviewutils/CoreRecyclerViewAdapter.java) is an abstract base class for using an adapter of objects with a given type. By using a generic type, we were able to override many boilerplate methods such as:

* add
* remove
* swapItems
* onBindViewHolder

Thanks to this handy utils class, it cuts down on a ton of boilerplate code inside your adapter, and make it very simple:

```java
   public class AccountAdapter extends CoreRecyclerViewAdapter<Account, AccountAdapter.AccountViewHolder>{
      public AccountAdapter(Context context, List<Account> accounts) {
         super(context, accounts);
      }
    
      @Override
      public AccountViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
         return new AccountViewHolder(LayoutInflater.from(context).inflate(R.layout.list_item_account, parent, false));
      }
    
      public class AccountViewHolder extends CoreViewHolder<Account> {
         ...
      }
   }
```