---
layout: post
title: 파이썬에서 R 사용하기
date: 2021-12-19
excerpt: "How to use R on Python"
tags: [R], [Rpy2]
comment: true
---



회사에서 ML/DL 이외에도 통계 모델들을 이용해서 문제를 푸는 경우도 있다. 이번에 R을 파이썬 환경에서 사용할 필요를 느껴서 그 방법을 정리하려고 한다. 동일 알고리즘을 파이썬보다 R을 사용했을 때, CPU 코어 사용량은 1/40로 줄어들었고 학습 시간은 50% 감소했다.



먼저 R을 설치한다.



```bash
# 오프라인 설치 매뉴얼로 작성

# yumdownloader를 이용해 오프라인 설치 시 필요한 dependency까지 한 번에 설치하기 위함

sudo yum install yum-utils



# 디펜던시 리스트 확인

yum deplist R-core-3.6.0-1.el7.x86_64.rpm

yum deplist R-core-devel-3.6.0-1.el7.x86_64.rpm

yum deplist R-java-devel-3.6.0-1.el7.x86_64.rpm

yum deplist R-devel-3.6.0-1.el7.x86_64.rpm

yum deplist R-java-3.6.0-1.el7.x86_64.rpm

yum deplist R-3.6.0-1.el7.x86_64.rpm



# 디펜던시까지 다운로드

yumdownloader --downloadonly --resolve R-core-3.6.0-1.el7.x86_64.rpm

yumdownloader --downloadonly --resolve R-core-devel-3.6.0-1.el7.x86_64.rpm

yumdownloader --downloadonly --resolve R-java-devel-3.6.0-1.el7.x86_64.rpm

yumdownloader --downloadonly --resolve R-devel-3.6.0-1.el7.x86_64.rpm

yumdownloader --downloadonly --resolve R-java-3.6.0-1.el7.x86_64.rpm

yumdownloader --downloadonly --resolve R-3.6.0-1.el7.x86_64.rpm



# 패키지 설치

yum install tre-common-0.8.0-18.20140228gitc2f5d13.el7.noarch.rpm

yum install tre-devel-0.8.0-18.20140228gitc2f5d13.el7.x86_64.rpm

yum install libRmath-devel-3.6.0-1.el7.x86_64.rpm

yum install R-core-3.6.0-1.el7.x86_64.rpm

yum install tre-0.8.0-18.20140228gitc2f5d13.el7.x86_64.rpm

yum install libRmath-3.6.0-1.el7.x86_64.rpm

yum install openblas-Rblas-0.3.3-2.el7.x86_64.rpm



yum install pcre2-devel-10.23-2.el7.x86_64.rpm

yum install texinfo-tex-5.1-5.el7.x86_64.rpm

yum install tre-devel-0.8.0-18.20140228gitc2f5d13.el7.x86_64.rpm

yum install R-core-devel-3.6.0-1.el7.x86_64.rpm



yum install R-java-devel-3.6.0-1.el7.x86_64.rpm



yum install R-devel-3.6.0-1.el7.x86_64.rpm



yum install R-java-3.6.0-1.el7.x86_64.rpm



yum install R-3.6.0-1.el7.x86_64.rpm



# R 설치 확인

rpm -qa | grep ^R

```



그 다음 Rpy2 패키지를 설치해서 파이썬에서 R을 사용할 수 있도록 한다. 디펜던시 패키지까지 온라인으로 한 번에 받은 다음, 반입할 CD에 담는다.

```bash
(base) conda activate {테스트 가상환경이름}

(테스트 가상환경이름) pip install rpy2

(테스트 환경이름) cd /home/{계정명}/anaconda3/envs/{가상환경이름}/lib/python3.6/site-packages



# site-packages 밑에 있는 패키지들 중 필요한 패키지들을 담는다. 다음 패키지들이 필요하다.

backports/

backports.zoneinfo-0.2.1.dist-info/

cffi/

cffi-1.15.0.dist-info/

_cffi_backend.cpython-36m-x86_64-linux-gnu.so/

cffi.libs/

_rinterface_cffi_abi.py

_rinterface_cffi_api.abi3.so

importlib_resources/

importlib_resources-5.4.0.dist-info/

jinja2/

Jinja2-3.0.3.dist-info/

markupsafe/

MarkupSafe-2.0.1.dist-info/

pycparser/

pycparser-2.21.dist-info/

pytz/

pytz-2021.3.dist-info/

pytz_deprecation_shim/

pytz_deprecation_shim-0.1.0.post0.dist-info/

rpy2/

rpy2-3.4.5.dist-info/

tzdata/

tzdata-2021.5.dist-info/

tzlocal/

tzlocal-4.1.dist-info/

zipp-3.6.0.dist-info/

zipp.py

```



파이썬에서 R을 사용하는 방법은 다음과 같다. pandas에서 데이터 전처리를 다 한 다음에, Fitting 및 Prediction만 R을 이용하는 방식으로 접근한다.

```python
import rpy2.robjects as ro

from rpy2.robjects.packages import importr

from rpy2.robjects import pandas2ri

from rpy2.robjects.conversion import localconverter



pandas2ri.activate()

importr('splines')



## pandas_df : pandas dataframe

## R_resolution_df : R dataframe

## R_dmin_df : R dataframe

with localconverter(ro.default_converter + pandas2ri.converter):

    R_pandas_df = ro.conversion.py2rpy(pandas_df)

r_formula = f"{feat} ~ bs(dmin, df=24)"

model = ro.r.lm(ro.Formula(r_formula), data=R_pandas_df)

predict_value = ro.r.predict(model, newdata = R_df, interval="prediction", level=0.99)

pred_df = pd.DataFrame(predict_value, columns=["pred", "lower", "upper"])

```

