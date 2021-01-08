
## total-shop-for-main-account:

SELECT orders, quantity, buyers, chat_conversion_rate, quantity_per_buyers 
FROM shopee_webchat_metrics_main_overview_tab WHERE main_account_id=${p.main_account_id} AND dt=${p.dt}



## each-shop-for-main-account: 

select si.id as shop_id,
    ifnull(sm.orders,0) orders,
    ifnull(sm.quantity,0) quantity,
    ifnull(sm.sales,0.0) sales,
    ifnull(sm.buyers,0) buyers,
    ifnull(sm.chat_conversion_rate,0.0) chat_conversion_rate,
    ifnull(sm.sales_per_quantity,0.0) sales_per_quantity,
    ifnull(sm.quantity_per_buyers,0.0) quantity_per_buyers,
    ifnull(sm.sales_per_buyers,0.0) sales_per_buyers
from (
    SELECT cast(SUBSTRING_INDEX(
        SUBSTRING_INDEX(ids,',',uid),
            ',',
            -1
            ) as signed) id
    from helper_uid_tab h
    left join (
    select '${p.shop_ids_str}' ids
    ) a on 1=1
    where h.uid <= ${p.shop_ids_cnt}
) si left join (
    SELECT shop_id, orders, quantity, sales, buyers, chat_conversion_rate, sales_per_quantity, quantity_per_buyers, sales_per_buyers 
    FROM shopee_webchat_metrics_main_shop_overview_tab 
    WHERE main_account_id=${p.main_account_id} AND dt=${p.dt} AND shop_id in ${p.shop_ids}
) sm on si.id=sm.shop_id;


## each-agent-for-main-account:

select si.id as agent_id,
    ifnull(sm.orders,0) orders,
    ifnull(sm.quantity,0) quantity,
    ifnull(sm.buyers,0) buyers,
    ifnull(sm.chat_conversion_rate,0.0) chat_conversion_rate,
    ifnull(sm.quantity_per_buyers,0.0) quantity_per_buyers
from (
    SELECT cast(SUBSTRING_INDEX(
        SUBSTRING_INDEX(ids,',',uid),
            ',',
            -1
            ) as signed) id
    from helper_uid_tab h
    left join (
    select '${p.agent_ids_str}' ids
    ) a on 1=1
    where h.uid <= ${p.agent_ids_cnt}
) si left join (
    SELECT agent_id, orders, quantity, buyers, chat_conversion_rate, quantity_per_buyers 
    FROM shopee_webchat_metrics_main_agent_overview_tab 
    WHERE main_account_id=${p.main_account_id} AND dt=${p.dt} AND agent_id in ${p.agent_ids}
) sm on si.id=sm.agent_id

## each-shop-for-agent:
select si.id as shop_id,
    ifnull(sm.orders,0) orders,
    ifnull(sm.quantity,0) quantity,
    ifnull(sm.sales,0.0) sales,
    ifnull(sm.buyers,0) buyers,
    ifnull(sm.chat_conversion_rate,0.0) chat_conversion_rate,
    ifnull(sm.sales_per_quantity,0.0) sales_per_quantity,
    ifnull(sm.quantity_per_buyers,0.0) quantity_per_buyers,
    ifnull(sm.sales_per_buyers,0.0) sales_per_buyers
from (
    SELECT cast(SUBSTRING_INDEX(
        SUBSTRING_INDEX(ids,',',uid),
            ',',
            -1
            ) as signed) id
    from helper_uid_tab h
    left join (
    select '${p.shop_ids_str}' ids
    ) a on 1=1
    where h.uid <= ${p.shop_ids_cnt}
) si left join (
    SELECT shop_id, orders, quantity, sales, buyers, chat_conversion_rate, sales_per_quantity, quantity_per_buyers, sales_per_buyers 
    FROM shopee_webchat_metrics_main_shop_agent_overview_tab WHERE main_account_id=${p.main_account_id} AND dt=${p.dt} AND shop_id in ${p.shop_ids} 
    AND agent_id=${p.agent_id}
) sm on si.id=sm.shop_id;

## each-agent-for-shop:

select si.id as agent_id,
    ifnull(sm.orders,0) orders,
    ifnull(sm.quantity,0) quantity,
    ifnull(sm.sales,0.0) sales,
    ifnull(sm.buyers,0) buyers,
    ifnull(sm.chat_conversion_rate,0.0) chat_conversion_rate,
    ifnull(sm.sales_per_quantity,0.0) sales_per_quantity,
    ifnull(sm.quantity_per_buyers,0.0) quantity_per_buyers,
    ifnull(sm.sales_per_buyers,0.0) sales_per_buyers
from (
    SELECT cast(SUBSTRING_INDEX(
        SUBSTRING_INDEX(ids,',',uid),
            ',',
            -1
            ) as signed) id
    from helper_uid_tab h
    left join (
    select '${p.agent_ids_str}' ids
    ) a on 1=1
    where h.uid <= ${p.agent_ids_cnt}
) si left join (
    SELECT agent_id, orders, quantity, sales, buyers, chat_conversion_rate, sales_per_quantity, quantity_per_buyers, sales_per_buyers 
    FROM shopee_webchat_metrics_main_shop_agent_overview_tab WHERE main_account_id=${p.main_account_id} AND dt=${p.dt} AND agent_id in ${p.agent_ids}
    AND shop_id=${p.shop_id}
) sm on si.id=sm.agent_id




## shop-real-overview:

* chat open , close: 

select shop_id
        ,count(distinct if(is_open,conversation_id,null)) chats_open_count
        ,count(distinct if(is_close,conversation_id,null)) chats_close_count
    from (
        select user_id account_id
            ,shop_id
            ,conversation_id
            ,if(conversation_status=0,1,0) is_open
            ,if(conversation_status=1 and last_close_time>#{p.today_ts},1,0) is_close
        from {kafka of webchat_conversation_analyze_tab}
        where 
         mtime > (CURRENT_DATE() - interval 30 day) {flink 数据窗口, 三十天, 存在哪里, 三十天, 目前表数据量三千多万条, 2.57GB}
           and user_id > 0
    ) agent
    GROUP BY shop_id

 schema: shopid,account_id,chats_open_count,chats_close_count


* chat_pending: 

select shop_id
        ,count(1) chats_pending_count
   from webchat_pending_analyze_tab
   where 
      pending_status = 1
   group by shop_id

   schema: shop_id,chats_pending_count, 历史全量, 好像最多是两年前的, 存 hbase, 目前表数据量六千多万, 2.78gb


* online_agent_count:

   select shop_id
        ,count(1) online_agents_count
    from webchat_agent_shop_relation_list_tab r
    inner join webchat_agent_chat_analyze_tab a
       on a.account_id=r.account_id
    where online_status=0
       and heartbeat_time>(UNIX_TIMESTAMP()-180)
       and r.agent_status=1
    group by shop_id

    schema: shopid,account_id,online_agents_count




select
    sum(ifnull(sm.chats_open_count,0)) chats_open_count,
    sum(ifnull(sm.chats_close_count,0)) chats_close_count,
    sum(ifnull(pd.chats_pending_count,0)) chats_pending_count,
    sum(ifnull(ht.online_agents_count,0)) online_agents_count
from (
    select shop_id
        ,count(distinct if(is_open,conversation_id,null)) chats_open_count
        ,count(distinct if(is_close,conversation_id,null)) chats_close_count
    from (
        select user_id account_id
            ,shop_id
            ,conversation_id
            ,if(conversation_status=0,1,0) is_open
            ,if(conversation_status=1 and last_close_time>#{p.today_ts},1,0) is_close
        from webchat_conversation_analyze_tab
        where shop_id in ${p.shop_ids}
           and mtime > (CURRENT_DATE() - interval 30 day)
           and user_id > 0
    ) agent
    GROUP BY shop_id
) sm
left join (
   select shop_id
        ,count(1) chats_pending_count
   from webchat_pending_analyze_tab
   where shop_id in ${p.shop_ids}
     and pending_status = 1
   group by shop_id
) pd on sm.shop_id=pd.shop_id
left join (
    select shop_id
        ,count(1) online_agents_count
    from webchat_agent_shop_relation_list_tab r
    inner join webchat_agent_chat_analyze_tab a
       on a.account_id=r.account_id
    where online_status=0
       and heartbeat_time>(UNIX_TIMESTAMP()-180)
       and r.agent_status=1
    group by shop_id
) ht on sm.shop_id=ht.shop_id;





> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTY5OTU1ODM5OSwtMjU1MzA4NjM1XX0=
-->