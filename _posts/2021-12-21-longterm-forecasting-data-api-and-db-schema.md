---
layout: post
title: 장기 부하 예측 데이터 API 및 DB 스키마 정의
date: 2021-12-21
excerpt: "Data API and DB Schema of Long Term Forecasting"
tags: [long term forecasting]
comment: true
---

# 데이터 API 정의
장기 부하 예측은 일단위 데이터를 집계한 데이터를 사용한다. WAS, DB, OS 그리고 E2E 트랜잭션/거래코드에 대한 장기 추이를 추정한다. 시간과 분석 대상에 대한 정보, 그리고 각 지표별 값을 일단위 집계해서 CSV로 넘겨받는다.
```
# WAS, DB, OS에서 target_id는 APM과 DPM에서 부여한 인스턴스 ID 및 Hostname을 ,사용
time,target_id,instance_metric1, instance_metric2, instance_metric3, ...., instance_metric(n-1), instance_metric(n)
2019-07-21 00:00:00,id_hostname,0.06657842115640085,0.04258315261778916, ... 300.46523437499997,741.0666666666666
2019-07-21 00:00:00,id_hostname,0.06657842115640085,0.04258315261778916, ... 300.46523437499997,741.0666666666666
2019-07-22 00:00:00,id_hostname,0.06657842115640085,0.04258315261778916, ... 300.46523437499997,741.0666666666666
2019-07-22 00:00:00,id_hostname,0.06657842115640085,0.04258315261778916, ... 300.46523437499997,741.0666666666666
2019-07-23 00:00:00,id_hostname,0.06657842115640085,0.04258315261778916, ... 300.46523437499997,741.0666666666666
2019-07-23 00:00:00,id_hostname,0.06657842115640085,0.04258315261778916, ... 300.46523437499997,741.0666666666666

#E2E에서는 트랜잭션이나 거래코드를 모니터링하므로 인스턴스 ID가 들어오지 않고 거래코드가 트랜잭션 정보가 들어오게 된다.
time,tx_code,txn,e2e_metric_1,e2e_metric_2
2019-07-21 00:00:00,tx_code_info,txn_info,94.0,94.0
2019-07-22 00:00:00,tx_code_info,txn_info,4034.0,4034.0
2019-07-31 00:00:00,tx_code_info,txn_info,47.0,47.0
2019-08-01 00:00:00,tx_code_info,txn_info,218422.0,218422.0
2019-08-02 00:00:00,tx_code_info,txn_info,217103.0,217103.0
2019-08-03 00:00:00,tx_code_info,txn_info,274113.0,274113.0
2019-08-04 00:00:00,tx_code_info,txn_info,258670.0,258670.0
```

# 테이블 스키마 정의
DB 테이블 스키마이다. UI에서 학습 데이터와 예측 결과를 조회할 수 있어야 화면에 표시할 수 있으므로, 각각 필요한 테이블을 따로 정의한다.
```SQL
CREATE SEQUENCE seq_module_prediction_result_table
 
CREATE TABLE module_predicttion_result_table(
    predict_id bigint NOT NULL,
    sys_id integer NOT NULL,
    type character varying(300) NOT NULL,
    target_id character varying(300),
    inst_id integer,
    predict_day character varying(8) NOT NULL,
    training_day_start character varying(8) NOT NULL,
    training_day_end character varying(8) NOT NULL,
    stat_name character varying(100) NOT NULL,
    predict_values character varying(5000) NOT NULL,
    predict_upper character varying(5000) NOT NULL,
    predict_lower character varying(5000) NOT NULL,
    create_dt timestamp with time zone NOT NULL DEFAULT now(),
    CONSTRAINT module_prediction_pk UNIQUE (inst_id, predict_day, training_day_start, training_day_end, stat_name)
);
COMMENT ON TABLE public.module_predicttion_result_table IS '장기예측모델 예측값 적재';

-- Column comments

COMMENT ON COLUMN public.module_predicttion_result_table.predict_id IS 'predict_id';
COMMENT ON COLUMN public.module_predicttion_result_table.training_day_start IS '학습데이터 시작일';
COMMENT ON COLUMN public.module_predicttion_result_table.training_day_end IS '학습데이터 종료일';
COMMENT ON COLUMN public.module_predicttion_result_table.predict_day IS '예측시작일';
COMMENT ON COLUMN public.module_predicttion_result_table.sys_id IS 'aiops_system.sys_id';
COMMENT ON COLUMN public.module_predicttion_result_table.type IS 'aiops_instance.type';
COMMENT ON COLUMN public.module_predicttion_result_table.target_id IS 'aiops_instance.target_id';
COMMENT ON COLUMN public.module_predicttion_result_table.inst_id IS 'aiops_instance.inst_id';
COMMENT ON COLUMN public.module_predicttion_result_table.stat_name IS '지표명';
COMMENT ON COLUMN public.module_predicttion_result_table.predict_values IS '예측값';
COMMENT ON COLUMN public.module_predicttion_result_table.create_dt IS 'PG insert time';
 
 
CREATE TABLE public.module_training_data_table (
    seq bigserial NOT NULL,
    sys_id int4 NOT NULL,
    "type" varchar(300) NOT NULL,
    target_id varchar(300) NOT NULL,
    txn varchar(300) NULL,
    "name" varchar(100) NOT NULL,
    value float8 NOT NULL,
    "time" timestamp NOT NULL,
    create_dt timestamptz NOT NULL DEFAULT now(),

    CONSTRAINT aiops_module_longterm_pkey PRIMARY KEY (seq)
);
COMMENT ON TABLE publicmodule_training_data_table IS '장기부하예측 과거 데이터 적재';
COMMENT ON COLUMN public.module_training_data_table.seq IS 'seq';
COMMENT ON COLUMN public.module_training_data_table.sys_id IS 'aiops_system.sys_id';
COMMENT ON COLUMN public.module_training_data_table."type" IS 'aiops_system.inst_type';
COMMENT ON COLUMN public.module_training_data_table.target_id IS 'target_id or tx_code';
COMMENT ON COLUMN public.module_training_data_table.txn IS 'txn';
COMMENT ON COLUMN public.module_training_data_table."name" IS '지표명';
COMMENT ON COLUMN public.module_training_data_table."value" IS '지표값';
COMMENT ON COLUMN public.module_training_data_table."time" IS '지표값 발생시간';
COMMENT ON COLUMN public.module_training_data_table.create_dt IS 'create_dt';
```
