   explain select
        recruit0_.recruit_id as recruit_1_23_,
        recruit0_.created_at as created_2_23_,
        recruit0_.modified_at as modified3_23_,
        recruit0_.category as category4_23_,
        recruit0_.contacturi as contactu5_23_,
        recruit0_.content as content6_23_,
        recruit0_.deleted_recruit as deleted_7_23_,
        recruit0_.recruit_end as recruit_8_23_,
        recruit0_.finished_recruit as finished9_23_,
        recruit0_.member_id as member_14_23_,
        recruit0_.register_recruit_type as registe10_23_,
        recruit0_.recruit_start as recruit11_23_,
        recruit0_.title as title12_23_,
        recruit0_.view as view13_23_ 
    from
        recruit recruit0_ 
    where
        recruit0_.category='PROJECT'
        and (
            recruit0_.recruit_id in (
                select
                    recruitski1_.recruit_id 
                from
                    recruit_skill recruitski1_ 
                inner join
                    recruit recruit2_ 
                        on recruitski1_.recruit_id=recruit2_.recruit_id 
                where
                    recruitski1_.skill='Spring'
            )
        ) 
        and (
            recruit0_.recruit_id in (
                select
                    recruitlim3_.recruit_id 
                from
                    recruit_limitation recruitlim3_ 
                inner join
                    recruit recruit4_ 
                        on recruitlim3_.recruit_id=recruit4_.recruit_id 
                where
                    recruitlim3_.type='백엔드'
            )
        ) 
    order by
        recruit0_.recruit_id desc limit 5;

vs 

두 개의 쿼리를 별도로 fetch 한 후 
Set<Long> recruitId로 id값의 중복 제거 후 in절로 쿼리하는 것 성능 비교 
