
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
    from webchat_agent_shop_relation_list_tab(一千两百万, 460MB) r
    inner join webchat_agent_chat_analyze_tab(三百万条, 300MB) a
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



<select id="queryShopOpenCloseData" resultMap="webChatShopDataResultMap">  
    SELECT shop_id,  
            count(DISTINCT if(is_open, conversation_id, NULL)) as chats_open_count,  
            count(DISTINCT if(is_close, conversation_id, NULL)) as chats_close_count,  
            0 as chats_pending_count,  
            0 as online_agents_count  
        FROM  
          (SELECT shop_id,  
                  conversation_id,  
                  if(conversation_status=0, 1, 0) as is_open,  
                  if(conversation_status=1 AND last_close_time>#{p.today_ts},1, 0) as is_close  
           FROM webchat_conversation_analyze_tab  
           WHERE shop_id in   
             <foreach item="sId" collection="p.shop_ids" separator="," open="(" close=")">  
               #{sId}  
             </foreach>  
             AND mtime > (CURRENT_DATE() - interval 30 DAY)  
             AND user_id > 0 ) agent  
        GROUP BY shop_id  
        <if test="limit!=null">  
            limit #{limit}  
        </if>  
</select>  
  
<select id="queryShopPendingData" resultMap="webChatShopDataResultMap">  
    SELECT shop_id,  
           0 as chats_open_count,   
           0 as chats_close_count,  
           count(1) as chats_pending_count,  
           0 as online_agents_count  
        FROM webchat_pending_analyze_tab  
        WHERE shop_id in   
        <foreach item="sId" collection="p.shop_ids" separator="," open="(" close=")">  
          #{sId}  
        </foreach>  
          AND pending_status = 1  
        GROUP BY shop_id  
    <if test="limit!=null">  
        limit #{limit}  
    </if>  
</select>  
  
<select id="queryShopOnlineAgentData" resultMap="webChatShopDataResultMap">  
    SELECT shop_id,  
           0 AS chats_open_count,  
           0 AS chats_close_count,  
           0 AS chats_pending_count,  
           count(*) AS online_agents_count  
        FROM  
          (SELECT shop_id,  
                (heartbeat_time + 180 - UNIX_TIMESTAMP()) AS time_minus  
          FROM webchat_agent_shop_relation_list_tab r  
          INNER JOIN webchat_agent_chat_analyze_tab a ON a.account_id=r.account_id  
          WHERE online_status=0  
              AND shop_id in   
              <foreach item="sId" collection="p.shop_ids" separator="," open="(" close=")">   
                  #{sId}   
              </foreach>  
              AND r.agent_status=1) sub_tab  
        WHERE sub_tab.time_minus > 0  
        GROUP BY sub_tab.shop_id  
    <if test="limit!=null">  
        limit ${limit}  
    </if>  
</select>  
  
<resultMap id="webChatShopDataResultMap" type="io.shopee.datagroup.dataway.entity.WebChatShopData">  
    <result property="shopId" column="shop_id"/>  
    <result property="chatsOpenCount" column="chats_open_count"/>  
    <result property="chatsCloseCount" column="chats_close_count"/>  
    <result property="chatsPendingCount" column="chats_pending_count"/>  
    <result property="onLineAgentsCount" column="online_agents_count"/>  
</resultMap>  
  
<select id="queryAccountOpenCloseUnreadData" resultMap="webChatAccountDataResultMap">  
    SELECT user_id AS account_id,   
           SUM(IF(conversation_status = 0 AND status != 1, unread_status, 0)) AS chats_unread_count,  
           SUM(IF(conversation_status = 0 AND status != 1, 1, 0)) AS chats_open_count,  
           SUM(IF(conversation_status = 1 AND status != 1 AND last_close_time > #{p.today_ts}, 1, 0)) AS chats_close_count,  
           0 as online_status,  
           0 as online_time_count,  
           0 as hangup_time_count  
    FROM webchat_conversation_analyze_tab        
    WHERE user_id IN  
           <foreach item="sId" collection="p.account_ids" separator="," open="(" close=")">  
             #{sId}  
           </foreach>        
          AND mtime > CURRENT_DATE() - INTERVAL 30 DAY        
    GROUP BY user_id   
    <if test="limit!=null">  
        limit ${limit}  
    </if>  
</select>  
  
<select id="queryAccountOnlineStatusData" resultMap="webChatAccountDataResultMap">  
    SELECT account_id,  
           0 as chats_open_count,  
           0 as chats_close_count,  
           0 as chats_unread_count,  
           IF(online_status != 2 AND heartbeat_time > unix_timestamp() - 180, online_status, 2) AS online_status,  
           IF(online_time >= #{p.today_ts}, online_time_count, 0) AS online_time_count,  
           IF(hangup_time >= #{p.today_ts}, hangup_time_count, 0) AS hangup_time_count        
    FROM  webchat_agent_chat_analyze_tab        
    WHERE account_id IN  
        <foreach item="sId" collection="p.account_ids" separator="," open="(" close=")">  
            #{sId}  
        </foreach>          
    <if test="limit!=null">  
        limit ${limit}  
    </if>  
</select>

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTM5MjY2ODA0NywtMjA0Mzc5NjYyOSwtNj
k5NTU4Mzk5LC0yNTUzMDg2MzVdfQ==
-->