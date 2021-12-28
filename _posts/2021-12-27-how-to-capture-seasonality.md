---
layout: post
title: 시계열에서 고정된 주기의 계절성 포착하기
date: 2021-12-27
excerpt: "How to capture seasonality in time series"
tags: [Time Series, Seasonality]
comment: true
---

# Seasonal Dummy Regression
시계열을 구성하는 요소는 크게 세 가지가 있는데, 각각 트렌드(Trend), 계절성(Seasonality) 그리고 나머지(Remainder)이다. 오늘은 고정된 주기의 시계열에서 계절성 진동을 포착하는 방법인 Seasonal Dummy Regression을 소개하려고 한다.

## Dataset
- 거래집중일을 만들기 위해, 데모 모니터링 시스템의 부하 패턴을 1주일에 하루는 부하가 평소보다 많도록 설계했다.

- 실제로 일별 집계 데이터를 보면, 일주일에 하루는 값이 튀는 것을 확인할 수 있다. 지표별로 분단위 값을 모아서 하루 단위로 평균 낸 값이다.

- ![business_day](/imgs/business_day.PNG){: width="50%"}{: .center}


## 아이디어
- 반복되는 주기를 사전에 알고 있거나 찾아낸 다음에, 주기에 해당하는 만큼의 더미변수를 만든다. 따라서 더미변수의 차원은 주기와 같다.

- 그리고 최소제곱법을 이용해서 모형을 적합한다.

- 더미변수는 표준기저이기 때문에 직교하므로, 다중공선성 위험에서 회피할 수 있다.

- 1주일 주기인 경우, 추정된 더미변수별 회귀계수는 요일별 값들의 평균이 된다.


- 예측할 때에는 예측할 날짜의 요일을 구해서 더미변수로 변환시킨 값이 입력값이 된다.

## 모형
$$ {y}_{t}\quad =\quad {\beta}_{0}\quad +\quad {\beta}_{1}{d}_{1,t}\quad +\quad {\beta}_{2}{d}_{2,t}\quad +\quad {\beta}_{3}{d}_{3,t}\quad +\quad{\beta}_{4}{d}_{4,t}\quad +\quad {\beta}_{5}{d}_{5,t}\quad +\quad {\beta}_{6}{d}_{6,t}\quad +\quad {\beta}_{7}{d}_{7,t}\quad +  \quad\varepsilon $$


- 원래는 Dummy Variable Trap을 피하기 위해 Category 전체 개수보다 한 개 적은 더미변수를 사용해야 하지만, 그렇게 되면 마지막 케이스는 예측값이 0으로 나오는 문제가 있어서 일부러 더미변수 개수를 줄이지 않았다.


## 코드
```python
    def learn_seasonality(self, data, metric_name):
        """
        Seasonal Dummy Regression을 이용해서 시계열 자료의 구성요소인 계절성을 탐지한다.

        :param data: 데이터셋
        :param metric_name: 지표 이름
        :return: 학습된 가변수 선형회귀 모형
        """
        self.logger.info(
            f"[learn_seasonality] start learning seasonality of {metric_name}"
        )

        seasonal_component = sm.tsa.seasonal_decompose(
            data[metric_name], model="additive"
        ).seasonal
        ''' 주기를 찾는 부분이므로, 다음 포스팅에서 설명 예정
        matrix_profile_period = self.get_repeated_subsequence(
            list(seasonal_component.values)
        )
        if matrix_profile_period is not None:
            top_seasonality_period = matrix_profile_period
        else:
            seasonality_information = self.learn_periods_and_frequency(
                seasonal_component
            )
            top_seasonality_period = np.round(
                seasonality_information["period1"], 1
            ).astype(dtype="int32")
        weekday_number_list = list(range(top_seasonality_period))

        prediction_length = self.calculate_predict_day_range(data)
        total_data_length = len(data) + prediction_length
        count, remainder = divmod(total_data_length, len(weekday_number_list))

        if remainder == 0:
            weekday_number_index = weekday_number_list * count
        else:
            weekday_number_index = (
                weekday_number_list * count + weekday_number_list[:remainder]
            )

        self.seasonality_weekday_number[metric_name] = weekday_number_index
        data[DAILY] = weekday_number_index[: len(data)]
        '''
        dummies = sm.tools.tools.add_constant(
            pd.get_dummies(data[DAILY]), has_constant="add"
        )

        lm = sm.regression.linear_model.OLS(data[metric_name], dummies)
        lm_model = lm.fit()
        self.seasonality_rsquare[metric_name] = lm_model.rsquared

        self.logger.info("[learn_seasonality] finish learning seasonality")

        return lm_model

    def predict_seasonality(self, model, metric_name, string_timestamp, mse_mode=False, test_datetime_index=None):
        """
        seasonality 모델을 이용해서 계절성을 예측

        Parameters
        ----------
        model : seasonality 있는 모델
        metric_name : 지표명
        string_timestamp : 예측시점들의 타임스탬프
        mse_mode : mse 계산할 것인지 결정하는 인자
        test_datetime_index : mse 계산시 입력되는 테스트 데이터의 타임스탬프

        Returns
        -------

        """
        self.logger.info("[predict_seasonality] start predicting seasonality")

        weekday_index = self.seasonality_weekday_number[metric_name]
        if mse_mode is True:
            self.logger.info("[predict_seasonality] calculating seasonality mse")
            forecast_datetime = test_datetime_index
            date_df = pd.DataFrame(
                weekday_index[len(weekday_index) - len(forecast_datetime) :],
                index=test_datetime_index,
                columns=["date_number"],
            )
        else:
            forecast_datetime = string_timestamp.map(
                lambda x: datetime.datetime.strptime(x, "%Y-%m-%d")
            )
            date_numbers = pd.Series(
                weekday_index[len(weekday_index) - len(forecast_datetime.index) :]
            )
            date_df = pd.concat([forecast_datetime, date_numbers], axis=1)
            date_df.columns = [TIME, "date_number"]
            date_df.set_index(TIME, inplace=True)

        dummies = sm.tools.tools.add_constant(
            pd.get_dummies(date_df["date_number"]), has_constant="add"
        )
        forecasted_seasonality = model.predict(dummies)

        self.logger.info("[predict_seasonality] finish predicting seasonality")

        return forecasted_seasonality
```

## 예측 결과 시각화
학습 데이터와 유사한 계절성을 예측 모델에서도 보여주고 있다.

![result](/imgs/result.PNG){: width="50%"}{: .center}


