# GPS 이동 경도를 지도에 그리기

수집한 GPS 정보(위도, 경도)로 지도에 이동 경로를 그려본다

코드 내용은 <b>[draw-a-route-on-the-map.ipynb](draw-a-route-on-the-map.ipynb)</b> 주피터 파일 내용을 참조

## 위치정보 수집
위도, 경도와 같은 GPS 위치정보는 별도의 외부 데이터를 사용하지 않고 직접 수집하여 사용하였다

Traccar를 사용하여 직접 위치 정보를 수집하였고, 별도의 많은 설정을 하지 않아도 쉽게 위치 정보를 수집할 수 있다

여기서는 기본 설정에서 위치 정보만 Database에 저장하여 사용하였다
- [Traccar - Modern GPS Tracking Platform](https://www.traccar.org/)
- [Traccar - Quick Start](https://www.traccar.org/quick-start/)
- [Traccar - Client](https://www.traccar.org/client/)


## 위치 정보 전처리
Traccar에 저장되는 Speed 값은 km/h가 아니라 knots 단위로 저장되므로, 단위를 변환해주었다
```python
df['speed'] = df['speed'].apply(lambda x : x * 1.852)
```
devicetime이 UTC로 저장되어, KST로 변환해주었다
```python
df['devicetime'] = df['devicetime'].apply(lambda x: x + timedelta(hours=9))
```
## 데이터 분할
하루치 이동 경로를 모두 가져와 처리하였는데, 총 세번의 이동 경로를 가진다(아침/저녁/야간)

개별 이동경로에 따라 그리기 위해서 시간 차로 데이터를 분할하였다

```python
# 데이터를 자르기 위해 데이터간의 시간차를 구한다
# 시간차가 1시간 이상인 경우, 별도의 경로로 구분하여 자른다
df['diffTime'] = df['devicetime'].diff()
df['diffTime'] = df['diffTime'].fillna(pd.Timedelta('00:00:00'))

# 1시간 이상 차이가 나는 데이터를 구한다
split_time_df = df['diffTime'] > pd.Timedelta(1, unit='h')

# 1시간 이상 차이가나는 데이터의 인덱스를 가져온디
split_idx = list(df[split_time_df].index)
split_idx

# 첫번째 이동 경로의 Dataframe을 필터링한다
route1 = df[:split_idx[0]]

# 두번째 이동 경로의 Dataframe을 필터링한다
route2 = df[split_idx[0]+1:split_idx[1]]

# 세번째 이동 경로의 Dataframe을 필터링한다
route3 = df[split_idx[1]+1:]
```

## 이동 경로 그리기
분할한 dataframe의 위치정보를 가지고 이동 경로를 그린다

이동 경로를 그리는 부분만을 따로 함수로 작성하였다
```python
def draw_location_to_map(loc_df: pd.DataFrame):
    """
    이동경로가 담긴 Dataframe으로 이동경로를 그려 객체를 반환한다
    
    :param df:
    :return:
    """
    # latitude, longitude를 리스트로 변경한다
    locations = [[loc_df.loc[i, 'latitude'], loc_df.loc[i, 'longitude']] for i in loc_df.index]

    # 첫번째 위치를 지도에 표시한다
    route_map = folium.Map(locations[0], tiles='Cartodb Positron', zoom_start=12)
    
    # 차량의 속도를 컬러로 표시하기 위한 컬러 맵을 생성한다
    colormap = linear.RdYlGn_11.scale(float(loc_df['speed'].min()), float(loc_df['speed'].max())).to_step(10)
    colormap.caption = 'Speed'
    colormap.add_to(route_map)

    # 차량의 속도를 컬러로 표시하기 위해, speed 컬럼을 리스트로 생성한다
    speeds = loc_df['speed'].astype('float').to_list()
    
    # 위치 정보를 기준으로 지도에 이동경로를 그린다
    nope = folium.ColorLine(locations, colors=speeds, colormap=colormap, weight=4).add_to(route_map)
    
    return route_map
```

## 결과
![draw_map](https://user-images.githubusercontent.com/31076511/141807855-2537d768-5a59-47c3-8391-e49f97f16d5b.png)