---
{"author":"jx2lee","aliases":"dbt 증분모델에서 merge 시 풀스캔 현상(with datetime partition column)","created":"2024-11-17T12:02:26.000+09:00","last-updated":"2024-09-13 21:17","tags":["dbt","incremental_model","finops"],"dg-publish":true,"dg-home-link":true,"dg-show-local-graph":true,"dg-show-backlinks":true,"dg-show-toc":false,"dg-show-inline-title":true,"dg-show-file-tree":false,"dg-enable-search":true,"dg-link-preview":true,"dg-show-tags":true,"dg-pass-frontmatter":false,"permalink":"/data/dbt/__/dbt-incremental-full-scan/","dgHomeLink":true,"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgEnableSearch":true,"dgLinkPreview":true,"dgShowTags":true,"dgPassFrontmatter":true,"noteIcon":""}
---


> [!info] incremental model 에서 merge 쿼리 시 풀 스캔 발생 문제를 살펴보고 해결방안을 공유해요.

- [[#Problem|Problem]]
- [[#Caused by|Caused by]]
- [[#Resolutions|Resolutions]]



### Problem
주문 원장 모델에 사용하는 prep_cum_traded_quantity 모델에서 billed_gb 가 830G 가 넘는 비용이 발생하였습니다. prep_cum_traded_quantity 전체 조회 시 발생하는 billed_gb 는 대략 800G 입니다.
```
23:05:12  54 of 68 OK created sql incremental model market_monitoring.prep_cum_traded_quantity  [SCRIPT (838.8 GB processed) in 42.47s]
```


### Caused by
insert_overwrite 전략으로 incremental 모델 실행 시 merge 쿼리에서 풀스캔이 발생했습니다.
- 실행쿼리(merge statement)는 다음과 같습니다.
    - 예상 billed_gb 는 233MB 이지만, 실제 실행 시 800G 를 사용합니다. 사용한 용량을 확인하면 prep_cum_traded_quantity 의 Total logical bytes 과 유사합니다 -> 풀스캔이 발생합니다.
    ```sql
        merge into `{placeholder}`.`cell_data`.`prep_cum_traded_quantity` as DBT_INTERNAL_DEST
            using (
            select
            * from `{placeholder}`.`cell_data`.`prep_cum_traded_quantity__dbt_tmp`
          ) as DBT_INTERNAL_SOURCE
            on FALSE
    
        when not matched by source
             and datetime_trunc(DBT_INTERNAL_DEST.created_at_kst, day) in unnest(dbt_partitions_for_replacement) 
            then delete
    
        when not matched then insert
            (`id`, `product_currency_code`, `created_at_kst`, `cum_traded_quantity`)
        values
            (`id`, `product_currency_code`, `created_at_kst`, `cum_traded_quantity`)
    ```
- 파티션 컬럼 타입은 datetime 으로 when not matched by source 라인에 존재하고, datetime_trunc 으로 파티션 프루닝이 정상적으로 동작해야하지만 **동작하지 않고 있습니다.**
    - 위 쿼리에서 datetime_trunc -> date_trunc 로 변경하면 파티션 프루닝이 동작합니다. 300MB 정도 사용해요.
    - `datetime 파티션 컬럼 -> 왜 datetime_trunc 로 파티션을 태우지 않지?` 에 대한 자료를 찾아볼 수 없었어요.. 서포트에 케이스 오픈해보고 싶네요. 저만 못찾는 걸까요?
- 관련 dbt-bigquery 링크
    - issue: https://github.com/dbt-labs/dbt-bigquery/issues/708
    - pull request: https://github.com/dbt-labs/dbt-bigquery/pull/993


### Resolutions
**{project-1} / {project-2} 에 사용하는 증분 모델 중 파티션 컬럼 타입이 datetime 인 모델은 prep_cum_traded_quantity 만 존재합니다.** 대부분의 증분모델들 파티션 타입은 **timestamp 혹은 date(주로 mart_ 모델)** 로 구성되어 있어요.

**따라서 이번에 찾은 문제는 이상거래감시시스템 원장 모델에만 발생하고 있습니다.** 해결할 수 있는 방법으로는 다음 두 가지가 있는데요, 1번으로 진행하는게 안전하다 생각합니다.

1. prep_cum_traded_quantity 모델의 파티션 컬럼 수정 (datetime -> date)
    - 파티션컬럼 수정도 해야하고 prep_all_order_report 수정도 필요합니다.
2. insert_overwrite 전략의 증분모델에 `copy_parittions` 옵션 사용하기
    - copy_partitions 옵션에 대한 자세한 정보는 [링크](https://docs.getdbt.com/reference/resource-configs/bigquery-configs#copying-partitions)를 확인해주세요. merge 쿼리 대신 bq cp 커맨드를 사용해서 파티션을 overwrite 하는 기능이에요. 단점으로 비용 혹은 쿼리 추적이 어렵다는 정도로 설명하고 있습니다.

### Conclusion
datetime 컬럼으로 파티션이 설정된 증분 모델에서 풀스캔이 발생하는 문제를 살펴봤습니다. 다행히도 많은 모델 중 하나의 모델에서만 datetime 컬럼이 파티션으로 설정되어 있었어요. 직관적으로도 datetime_trunc 로 파티션 프루닝이 발생하지 않는게 의아했지만, GCP 에서 지원을 안하는 이유가 있지 않을까 생각했어요.

- datetime 컬럼으로 파티션 프루닝을 태울때는 date 함수를 사용해주세요.
- 직관적이지 않아 조금 의아했지만 파티션 컬럼 설정 시 `timestamp` 로 통일하는게 마음이 편할 것 같다 생각했어요.
