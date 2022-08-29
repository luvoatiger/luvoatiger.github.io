# 논문 리뷰 (3) : Bayesian Time Series Forecasting with Change Point and Anomaly Detection - class implementation

## 개요

이전 버전에서는 ipython notebook 형식으로 코드를 작성했다. 다만, 실제 프로덕트 레벨에 반영하려면 클린 코드나 확장성 등을 고려한 디자인 등 조금 더 생각을 해야 한다. 그래서 이번에는 클래스 형태로 변경한 코드를 업로드한다. 내용은 기존과 동일하다.

## 전체 코드

### 라이브러리 임포트

```python
import copy
import math
from collections import Counter

import numpy as np
import pandas as pd
import statsmodels.api as sm
import tensorflow.compat.v2 as tf
tf.enable_v2_behavior()
import tensorflow_probability as tfp
from tensorflow_probability import distributions as tfd
from scipy import stats
import pykalman
```

### 메인 클래스 작성
- 사용처에서 호출하는 부분이다. 호출할 때, 사용할 탐지 방법과 알고리즘에서 필요한 인자를 넘겨주도록 작성했다.
- 메인 클래스에서는 내부적으로 그에 맞는 알고리즘을 생성해준다.
- 참고로 HiddenMarkovModel 기반 방법은 이번 포스팅과는 관련 없으므로, 추후에 작성할 계획이다.
```python
class BayesianDetector():
    def __init__(self, config, logger, method="hmm"):
        self.config = config
        self.logger = logger
        self.method = method
        if self.method == "hmm":
            self.bayesian_detector = HiddenMarkovModelDetector(config, logger)
        else:
            self.bayesian_detector = StateSpaceModelDetector(config, logger)
 
    def detect(self, data):
        self.bayesian_detector.detect(data)
```

### 알고리즘 작성
#### (1) 파라미터 클래스 작성
- 파라미터들과 파라미터 업데이트 부분을 하나로 묶고, 파라미터 기반 계산 로직들을 같이 묶는 방식으로 서로를 분리했다. 
- 두 로직이 섞이면 값을 추적하기가 어렵고, 객체 지향 방식에서는 요청을 통해서 각 객체들이 서로 소통하는 방식으로 로직이 구성되기 때문이다.
- 
```python
class StateSpaceModelParameters():
    def __init__(self, config, logger):
        self.config = config
        self.logger = logger
 
 
    def init_param(self, data):
        self.seasonality_period = 7
        self.segment_length = 7
        self.series_length = len(data)
        self.indicate_prior_anomaly_points()
        self.indicate_prior_change_points()
 
        self.initial_alpha = [np.mean(data[:self.seasonality_period]), self.initial_delta] + self.initial_gamma
        self.standard_deviations = {key: np.std(data) for key in ['e', 'o', 'r', 'w', 'v', 'u']}
        self.generate_new_variances()
 
        self.anomaly_probability, self.change_probability = 1 / len(data), 1 / len(data)
 
 
    def update_initial_alpha(self, initial_hidden_variable, generated_smoothed_state_mean):
        initial_alpha = initial_hidden_variable[0:2] + list(generated_smoothed_state_mean[6][2:])
 
        return initial_alpha
 
    def update_standard_deviations(self, posterior_zc, posterior_za, seasonalities):
        not_anomaly = 0
        anomaly = 0
        for t, anomaly_point in enumerate(posterior_za):
            if anomaly_point == 0:
                not_anomaly += (self.generated_variances['e'][t] ** 2) / (t + 1)
            else:
                anomaly += (self.generated_variances['o'][t] ** 2) / (t + 1)
 
        if not_anomaly != 0:
            self.standard_deviations['e'] = math.sqrt(not_anomaly)
        if anomaly != 0:
            self.standard_deviations['o'] = math.sqrt(anomaly)
 
        not_chage_point = 0
        change_point = 0
        for t, change_point in enumerate(posterior_zc):
            if change_point == 0:
                not_chage_point += (self.generated_variances['u'][t] ** 2) / (t + 1)
            else:
                change_point += (self.generated_variances['r'][t] ** 2) / (t + 1)
 
        if not_chage_point != 0:
            self.standard_deviations['u'] = math.sqrt(not_chage_point)
        if change_point != 0:
            self.standard_deviations['r'] = math.sqrt(change_point)
 
        self.standard_deviations['v'] = math.sqrt(np.mean(np.square(self.generated_variances['v'])))
        gamma = 0
        for seasonality in seasonalities:
            gamma = sum(seasonality) ** 2
        self.standard_deviations['w'] = math.sqrt(gamma / len(seasonalities))
 
        return True
 
    def generate_new_variances(self):
        self.generated_variances = {}
        for key in self.standard_deviations.keys():
            self.generated_variances[key] = np.random.normal(loc=0, scale=self.standard_deviations[key], size=self.series_length)
 
        return True
 
    def indicate_prior_anomaly_points(self):
        self.prior_anomaly_indicator = np.random.binomial(size=self.series_length, n=1, p=self.anomaly_probability)
 
        return True
 
    def indicate_prior_change_points(self):
        self.prior_change_indicator = np.random.binomial(size=self.series_length, n=1, p=self.change_probability)
 
        return True
 
    def indicate_poseterior_anomaly_points(self, data):
        posterior_za_probability = []
        posterior_za = []
 
        for t in range(len(data)):
            if self.prior_anomaly_indicator[t] == 0:  # not anomaly point
                residual_square = self.generated_variances['e'][t] ** 2
            else:  # anomaly point
                residual_square = self.generated_variances['o'][t] ** 2
 
            # calculate posterior for each case
            anomaly_posterior = (self.anomaly_probability / self.standard_deviations['o']) * math.exp(
                -(residual_square / (2 * (self.standard_deviations['o'] ** 2))))
            normal_posterior = ((1 - self.anomaly_probability) / self.standard_deviations['e']) * math.exp(
                -(residual_square / (2 * (self.standard_deviations['e'] ** 2))))
 
            # weight average
            posterior_anomaly_prob = round(anomaly_posterior / (normal_posterior + anomaly_posterior), 3)
            posterior_za_probability.append(posterior_anomaly_prob)
            posterior_za.append(np.random.binomial(n=1, p=posterior_anomaly_prob, size=1)[0])
 
        return pd.Series(posterior_za)
 
 
    def indicate_poseterior_change_points(self, latent_mu):
        # initialize
        posterior_zc_probability = []
        posterior_zc = []
 
        for t in range(len(self.prior_change_indicator)):
            if self.prior_change_indicator[t] == 0:  # not change point
                residual_square = self.generated_variances['u'][t] ** 2
            else:  # change point
                residual_square = self.generated_variances['r'][t] ** 2
 
            # calculate posterior of each case
            change_posterior = (self.change_probability / self.standard_deviations['r']) * math.exp(
                -(residual_square / (2 * (self.standard_deviations['r'] ** 2))))
            normal_posterior = ((1 - self.change_probability) / self.standard_deviations['u']) * math.exp(
                -(residual_square / (2 * (self.standard_deviations['u'] ** 2))))
 
            # weight average
            posterior_change_prob = round(change_posterior / (normal_posterior + change_posterior), 3)
            posterior_zc_probability.append(posterior_change_prob)
            posterior_zc.append(np.random.binomial(n=1, p=posterior_change_prob, size=1)[0])
 
        # segment control
        posterior_zc = pd.Series(posterior_zc)
        posterior_zc = self._change_point_segment_control(posterior_zc, latent_mu, posterior_zc)
 
        return posterior_zc
 
 
    def _change_point_segment_control(self, posterior_zc, mu, change_point_index):
        # find change point index
        indicators = list(posterior_zc[posterior_zc == 1].index.values)
 
        for i in range(len(indicators))[1:]:
            # length between two change point is further than segment_length
            if indicators[i] - indicators[i - 1] >= self.segment_length:
                continue
            else: # length between two change point isn't further than segment_length
                # check differences between one-step before value and one-step ahead value
                if np.abs(mu[indicators[i - 1] - 1] - mu[indicators[i] + 1]) <= self.standard_deviations['r']/2:
                    posterior_zc[indicators[i - 1]] = 0
                    posterior_zc[indicators[i]] = 0
 
                else:
                    random_index = np.random.binomial(n=1, p=0.5, size=1)[0]
                    if random_index == 1:
                        posterior_zc[change_point_index[i - 1]] = 0
                    else:
                        posterior_zc[change_point_index[i]] = 0
 
        return posterior_zc
```

#### (2) 계산 부분 작성
```python
class StateSpaceModelDetector(BayesianDetector):
    def __init__(self, config, logger):
        self.config = config
        self.logger = logger
        self.parameters = StateSpaceModelParameters(config, logger)
 
 
    def detect(self, data):
        self.parameters.init_param()
        anomaly_points, change_points = self.detect_anomaly_and_change_points(data)
 
        return anomaly_points, change_points
 
 
    def forecast(self):
        # TODO need to be implemented
        pass
 
 
    def test_likelihood_convergence(self):
        # TODO need to research how to converge non-convex log likelihood function
        return False
 
 
    def calculate_log_likelihood(self, posterior_za, posterior_zc, seasonalities):
        # TODO likelihood가 0으로 계산되는 이슈 해결 필요
        log_likelihood = []
 
        for t, anomaly_point in enumerate(posterior_za):
            if anomaly_point == 0:
                likelihood = stats.norm.logpdf(0, loc=self.parameters.generated_variances['e'][t], scale=self.parameters.standard_deviations[
                    'e'])  # the likelihood g(x1, x2) in the paper is the form of normal density evaluated at 0 with mean x1, standard deviation x2
                log_likelihood.append(likelihood)
            else:
                likelihood = stats.norm.logpdf(0, loc=self.parameters.generated_variances['o'][t], scale=self.parameters.standard_deviations['o'])
                log_likelihood.append(likelihood)
            likelihood = stats.bernoulli.logpmf(k=anomaly_point, p=self.parameters.anomaly_probability)
            log_likelihood.append(likelihood)
 
        for t, change_point in enumerate(posterior_zc):
            if change_point == 0:
                likelihood = stats.norm.logpdf(0, loc=self.parameters.generated_variances['u'][t], scale=self.parameters.standard_deviations['u'])
                log_likelihood.append(likelihood)
            else:
                likelihood = stats.norm.logpdf(0, loc=self.parameters.generated_variances['r'][t], scale=self.parameters.standard_deviations['r'])
                log_likelihood.append(likelihood)
            likelihood = stats.bernoulli.logpmf(k=change_point, p=self.parameters.change_probability)
            log_likelihood.append(likelihood)
 
        for t, delta_variance in enumerate(self.parameters.generated_variances['v']):
            likelihood = stats.norm.logpdf(0, loc=delta_variance, scale=self.parameters.standard_deviations['v'])
            log_likelihood.append(likelihood)
 
        for t, seasonality in enumerate(seasonalities):
            likelihood = stats.norm.logpdf(0, loc=-np.sum(seasonality), scale=self.parameters.standard_deviations['w'])
            log_likelihood.append(likelihood)
 
        return np.exp(np.sum(log_likelihood))
 
 
    def apply_fake_path_trick(self, generated_data, real_data, mu):
        # TODO need to research
 
        ## Calculate $E(\tilde\alpha$ | \tildey)$ by applying kalman filter and kalman smoother to generated time series
        generated_data_filter = pykalman.KalmanFilter(initial_state_mean=self.parameters.initial_alpha[0], n_dim_obs=1)
        generated_filtered_state_mean, generated_filtered_state_covariance = generated_data_filter.em(
            generated_data).filter(generated_data)
        generated_smoothed_state_mean, generated_smoothed_state_covariance = generated_data_filter.em(
            generated_filtered_state_mean).smooth(generated_filtered_state_mean)
 
        ## Calculate $E($\alpha$ | $y$)$ by applying kalman smoother to real time series
        real_data_filter = pykalman.KalmanFilter(initial_state_mean = real_data[0], n_dim_obs=1)
        real_filtered_state_mean, real_filtered_state_covariance = real_data_filter.em(real_data).filter(real_data)
        real_smoothed_state_mean, real_smoothed_state_covariance = real_data_filter.em(real_filtered_state_mean).smooth(real_filtered_state_mean)
 
        ## sample alpha
        inferred_alpha = mu - generated_smoothed_state_mean + real_smoothed_state_mean
 
        return inferred_alpha, generated_smoothed_state_mean
 
 
    def detect_anomaly_and_change_points(self, data):
        is_likelihood_converged = self.test_likelihood_convergence()
 
        while not is_likelihood_converged:
            is_likelihood_converged = True # TODO 구현되면 제거 예정
 
            inferred_alpha, generated_data_smoothing, seasonalities = self.infer_alpha_vector(data)
            posterior_anomaly_points = self.parameters.indicate_poseterior_anomaly_points()
            posterior_change_points = self.parameters.indicate_poseterior_change_points()
            self.parameters.update_standard_deviations(posterior_change_points, posterior_anomaly_points, seasonalities)
            self.parameters.update_initial_alpha(inferred_alpha, generated_data_smoothing)
            evaluated_likelihood, is_likelihood_converged = self.calculate_log_likelihood(posterior_anomaly_points, posterior_change_points, seasonalities)
 
        return posterior_anomaly_points, posterior_change_points
 
 
    def infer_alpha_vector(self, real_data):
        self.parameters.indicate_prior_anomaly_points()
        self.parameters.indicate_prior_change_points()
        self.parameters.generate_new_variances()
        generated_data, generated_mu, delta, seasonalities = self.generate_time_series()
        inferred_alpha, generated_data_smoothing = self.apply_fake_path_trick(generated_data, real_data, generated_mu)
 
        return inferred_alpha, generated_data_smoothing, seasonalities
 
 
    def generate_time_series(self):
        mu_t, delta_t, seasonality = self.parameters.initial_alpha[0], self.parameters.initial_alpha[1], self.parameters.initial_alpha[2:]
        mu, delta, seasonalities, y = [mu_t], [delta_t], [seasonality], []
 
        alpha = [self.parameters.initial_alpha]
 
        for t in range(self.parameters.series_length):
            # evaluate mu and delta at t using transition equation
            if self.parameters.prior_change_indicator[t] == 0:
                mu_t = mu_t + delta_t + self.parameters.generated_variances['u'][t]
            else:
                mu_t = mu_t + delta_t + self.parameters.generated_variances['r'][t]
            delta_t = delta_t + self.parameters.generated_variances['v'][t]
 
            # evaluate seasonality at t using transition equation
            seasonality_t = -sum(seasonality[:t]) + self.parameters.generated_variances['w'][t]
            del seasonality[0]
            seasonality.append(seasonality_t)
            seasonalities.append(copy.deepcopy(seasonality))
 
            # evaluate y at t using observation equation
            if self.parameters.prior_anomaly_indicator[t] == 0:
                y_t = mu_t + delta_t + self.parameters.generated_variances['e'][t]
            else:
                y_t = mu_t + delta_t + self.parameters.generated_variances['o'][t]
 
            y.append(y_t)
            mu.append(mu_t)
            delta.append(delta_t)
            alpha.append([mu_t, delta_t] + seasonality)
 
        return y, mu, delta, seasonalities
```