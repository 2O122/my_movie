import streamlit as st
import requests
import pandas as pd
from datetime import datetime, timedelta
from zoneinfo import ZoneInfo


# --------------------------------------------------
# 1. 기본 설정
# --------------------------------------------------

st.set_page_config(
    page_title="어제의 박스오피스",
    page_icon="🎬",
    layout="wide"
)

st.title("🎬 어제의 박스오피스")
st.caption("영화관입장권통합전산망(KOBIS) 일별 박스오피스")


# --------------------------------------------------
# 2. 한국 시간 기준으로 '어제' 날짜 계산
# --------------------------------------------------
# 스트림릿 클라우드 서버의 시간이 한국 시간이 아닐 수 있기 때문에
# 서버의 현재 시간을 그대로 사용하지 않고 한국 시간(KST)을 사용합니다.

KST = ZoneInfo("Asia/Seoul")

now_kst = datetime.now(KST)
yesterday = now_kst.date() - timedelta(days=1)

# KOBIS API가 요구하는 날짜 형식: YYYYMMDD
target_date = yesterday.strftime("%Y%m%d")

# 화면에 보여줄 날짜 형식
display_date = yesterday.strftime("%Y년 %m월 %d일")


# --------------------------------------------------
# 3. KOBIS API에서 어제의 박스오피스 가져오기
# --------------------------------------------------

API_URL = (
    "https://www.kobis.or.kr/"
    "kobisopenapi/webservice/rest/boxoffice/"
    "searchDailyBoxOfficeList.json"
)


@st.cache_data(ttl=3600)
def get_boxoffice(target_dt):
    """
    KOBIS API에서 특정 날짜의 일별 박스오피스를 가져옵니다.

    @st.cache_data(ttl=3600)
    -> 같은 날짜를 다시 요청하면 약 1시간 동안
       저장해 둔 결과를 사용합니다.
       따라서 불필요하게 API를 반복 호출하지 않습니다.
    """

    # Streamlit Secrets에 저장한 API 인증키를 가져옵니다.
    # 실제 인증키를 코드에 직접 적지 않습니다.
    try:
        api_key = st.secrets["KOBIS_KEY"]
    except Exception:
        return {
            "success": False,
            "message": (
                "KOBIS_KEY를 찾을 수 없습니다.\n\n"
                "Streamlit Cloud의 Settings → Secrets에 "
                "`KOBIS_KEY`가 등록되어 있는지 확인하세요."
            ),
            "data": None
        }

    # KOBIS API에 보낼 요청 데이터
    params = {
        "key": api_key,
        "targetDt": target_dt
    }

    try:
        # API 호출
        response = requests.get(
            API_URL,
            params=params,
            timeout=10
        )

        # HTTP 오류가 발생하면 예외 발생
        response.raise_for_status()

        # JSON 형태로 변환
        result = response.json()

    except requests.exceptions.Timeout:
        return {
            "success": False,
            "message": (
                "KOBIS API 요청 시간이 초과되었습니다.\n\n"
                "잠시 후 다시 실행해 보세요."
            ),
            "data": None
        }

    except requests.exceptions.RequestException as e:
        return {
            "success": False,
            "message": (
                "KOBIS API에 연결하지 못했습니다.\n\n"
                "인터넷 연결이나 KOBIS API 주소를 확인해 주세요.\n\n"
                f"오류 내용: {e}"
            ),
            "data": None
        }

    except ValueError:
        return {
            "success": False,
            "message": (
                "KOBIS API에서 올바른 JSON 데이터를 받지 못했습니다.\n\n"
                "잠시 후 다시 시도해 주세요."
            ),
            "data": None
        }

    # --------------------------------------------------
    # 4. faultInfo 확인
    # --------------------------------------------------
    # KOBIS API는 인증키가 잘못되어도 HTTP 상태코드가 200으로
    # 올 수 있고, 이 경우 faultInfo가 들어옵니다.

    if "faultInfo" in result:
        fault_info = result["faultInfo"]

        fault_code = fault_info.get("faultCode", "알 수 없음")
        fault_message = fault_info.get(
            "message",
            "알 수 없는 오류가 발생했습니다."
        )

        return {
            "success": False,
            "message": (
                "KOBIS API에서 오류를 반환했습니다.\n\n"
                f"- 오류 코드: {fault_code}\n"
                f"- 오류 내용: {fault_message}\n\n"
                "특히 KOBIS_KEY가 올바른지 확인해 주세요."
            ),
            "data": None
        }

    # --------------------------------------------------
    # 5. 박스오피스 데이터 확인
    # --------------------------------------------------

    boxoffice_result = result.get("boxOfficeResult")

    if not boxoffice_result:
        return {
            "success": False,
            "message": (
                "KOBIS에서 박스오피스 결과를 찾을 수 없습니다.\n\n"
                "API 응답 구조나 조회 날짜를 확인해 주세요."
            ),
            "data": None
        }

    movie_list = boxoffice_result.get("dailyBoxOfficeList", [])

    # 영화 목록이 비어 있는 경우
    if not movie_list:
        return {
            "success": False,
            "message": (
                f"{display_date}의 영화 목록이 비어 있습니다.\n\n"
                "다음 사항을 확인해 주세요.\n"
                "1. KOBIS에 해당 날짜의 박스오피스 자료가 등록되었는지\n"
                "2. KOBIS API가 정상적으로 작동하는지\n"
                "3. 인증키가 정상적으로 등록되어 있는지"
            ),
            "data": None
        }

    return {
        "success": True,
        "message": "",
        "data": movie_list
    }


# --------------------------------------------------
# 6. API 실행
# --------------------------------------------------

result = get_boxoffice(target_date)


# --------------------------------------------------
# 7. API 요청 실패 시 안내
# --------------------------------------------------

if not result["success"]:
    st.error(result["message"])

    st.info(
        "💡 Streamlit Cloud에서 문제가 발생했다면 "
        "Settings → Secrets에 다음과 같이 등록되어 있는지 확인하세요.\n\n"
        "KOBIS_KEY = \"발급받은_인증키\""
    )

    st.stop()


# --------------------------------------------------
# 8. API 결과를 데이터프레임으로 변환
# --------------------------------------------------

movies = result["data"]

df = pd.DataFrame(movies)


# --------------------------------------------------
# 9. 숫자로 오는 값들을 실제 숫자형으로 변환
# --------------------------------------------------
# KOBIS API에서는 숫자도 문자열로 전달되기 때문에
# 그래프와 정렬을 제대로 사용하려면 숫자형으로 변환해야 합니다.

number_columns = [
    "rank",
    "rankInten",
    "audiCnt",
    "audiAcc",
    "scrnCnt",
    "showCnt"
]

for column in number_columns:
    if column in df.columns:
        df[column] = pd.to_numeric(
            df[column],
            errors="coerce"
        ).fillna(0).astype(int)


# --------------------------------------------------
# 10. 제목과 조회 날짜
# --------------------------------------------------

st.subheader(f"📅 {display_date} 박스오피스")

st.write(
    f"전날인 **{display_date}**에 집계된 영화관 박스오피스입니다."
)


# --------------------------------------------------
# 11. 1위 영화 정보
# --------------------------------------------------

if len(df) > 0:

    first_movie = df.iloc[0]

    st.subheader("🏆 오늘의 1위")

    st.markdown(
        f"## {first_movie['movieNm']}"
    )

    # 1위 영화의 주요 정보를 카드 3개로 표시
    card1, card2, card3 = st.columns(3)

    with card1:
        st.metric(
            "당일 관객수",
            f"{first_movie['audiCnt']:,}명"
        )

    with card2:
        st.metric(
            "누적 관객수",
            f"{first_movie['audiAcc']:,}명"
        )

    with card3:
        st.metric(
            "스크린수",
            f"{first_movie['scrnCnt']:,}개"
        )


# --------------------------------------------------
# 12. 관객수 상위 5편 그래프
# --------------------------------------------------

st.subheader("📊 관객수 상위 5편")

# 관객수 기준으로 내림차순 정렬
top5 = (
    df.sort_values(
        by="audiCnt",
        ascending=False
    )
    .head(5)
    .copy()
)

# 영화명을 그래프의 인덱스로 설정
top5_chart = top5[
    ["movieNm", "audiCnt"]
].set_index("movieNm")

# 막대그래프 표시
st.bar_chart(
    top5_chart,
    y="audiCnt",
    horizontal=False
)

st.caption("※ 당일 관객수 기준 상위 5편입니다.")


# --------------------------------------------------
# 13. 전체 박스오피스 표
# --------------------------------------------------

st.subheader("🎞️ 전체 박스오피스")

# 화면에 보여줄 컬럼만 선택
table_df = df[
    [
        "rank",
        "movieNm",
        "openDt",
        "audiCnt",
        "audiAcc",
        "scrnCnt"
    ]
].copy()


# 컬럼명을 한국어로 변경
table_df.columns = [
    "순위",
    "영화명",
    "개봉일",
    "관객수",
    "누적관객",
    "스크린수"
]


# 숫자에 천 단위 쉼표를 적용하기 위한 표시용 함수
def format_number(value):
    return f"{value:,}"


# 표에 표시할 숫자 컬럼
for column in ["순위", "관객수", "누적관객", "스크린수"]:
    table_df[column] = table_df[column].apply(format_number)


# 표 출력
st.dataframe(
    table_df,
    use_container_width=True,
    hide_index=True
)


# --------------------------------------------------
# 14. 데이터 출처
# --------------------------------------------------

st.caption(
    "데이터 출처: 영화관입장권통합전산망(KOBIS) "
    "일별 박스오피스 Open API"
)
```python
import streamlit as st
import requests
import pandas as pd
from datetime import datetime, timedelta
from zoneinfo import ZoneInfo


# --------------------------------------------------
# 1. 기본 화면 설정
# --------------------------------------------------

st.set_page_config(
    page_title="어제의 박스오피스",
    page_icon="🎬",
    layout="wide"
)

st.title("🎬 어제의 박스오피스")
st.caption("영화관입장권통합전산망(KOBIS) 일별 박스오피스")


# --------------------------------------------------
# 2. 한국 시간 기준으로 '어제' 날짜 계산
# --------------------------------------------------
# Streamlit Cloud 서버의 시간은 한국 시간이 아닐 수 있기 때문에
# 한국 시간(KST)을 기준으로 오늘 날짜를 구한 뒤 하루를 뺍니다.

KST = ZoneInfo("Asia/Seoul")

now_kst = datetime.now(KST)
yesterday = now_kst.date() - timedelta(days=1)

# KOBIS API에서 사용하는 날짜 형식
target_date = yesterday.strftime("%Y%m%d")

# 화면에 표시할 날짜
display_date = yesterday.strftime("%Y년 %m월 %d일")


# --------------------------------------------------
# 3. KOBIS API 주소
# --------------------------------------------------

API_URL = (
    "https://www.kobis.or.kr/"
    "kobisopenapi/webservice/rest/boxoffice/"
    "searchDailyBoxOfficeList.json"
)


# --------------------------------------------------
# 4. KOBIS API에서 박스오피스 데이터 가져오기
# --------------------------------------------------
# 같은 날짜를 다시 요청하면 1시간 동안 저장된 데이터를 사용합니다.

@st.cache_data(ttl=3600)
def get_boxoffice(target_dt):

    # Streamlit Secrets에서 API 인증키를 가져옵니다.
    # 실제 인증키는 코드에 적지 않습니다.
    try:
        api_key = st.secrets["KOBIS_KEY"]

    except Exception:
        return {
            "success": False,
            "message": (
                "KOBIS_KEY를 찾을 수 없습니다.\n\n"
                "Streamlit Cloud의 Settings → Secrets에 "
                "`KOBIS_KEY`가 등록되어 있는지 확인하세요."
            ),
            "data": None
        }

    # KOBIS API에 전달할 값
    params = {
        "key": api_key,
        "targetDt": target_dt
    }

    try:
        # API 요청
        response = requests.get(
            API_URL,
            params=params,
            timeout=10
        )

        # HTTP 오류 확인
        response.raise_for_status()

        # JSON 데이터로 변환
        result = response.json()

    except requests.exceptions.Timeout:
        return {
            "success": False,
            "message": (
                "KOBIS API 요청 시간이 초과되었습니다.\n\n"
                "잠시 후 다시 실행해 주세요."
            ),
            "data": None
        }

    except requests.exceptions.RequestException as e:
        return {
            "success": False,
            "message": (
                "KOBIS API에 연결하지 못했습니다.\n\n"
                "인터넷 연결이나 KOBIS API 주소를 확인해 주세요.\n\n"
                f"오류 내용: {e}"
            ),
            "data": None
        }

    except ValueError:
        return {
            "success": False,
            "message": (
                "KOBIS API에서 올바른 JSON 데이터를 받지 못했습니다.\n\n"
                "잠시 후 다시 시도해 주세요."
            ),
            "data": None
        }


    # --------------------------------------------------
    # 5. faultInfo 확인
    # --------------------------------------------------
    # KOBIS는 인증키가 잘못되어도 상태코드가 200일 수 있습니다.
    # 따라서 faultInfo가 있는지도 확인해야 합니다.

    if "faultInfo" in result:

        fault_info = result["faultInfo"]

        fault_code = fault_info.get(
            "faultCode",
            "알 수 없음"
        )

        fault_message = fault_info.get(
            "message",
            "알 수 없는 오류가 발생했습니다."
        )

        return {
            "success": False,
            "message": (
                "KOBIS API에서 오류를 반환했습니다.\n\n"
                f"- 오류 코드: {fault_code}\n"
                f"- 오류 내용: {fault_message}\n\n"
                "KOBIS_KEY가 올바르게 등록되어 있는지도 확인해 주세요."
            ),
            "data": None
        }


    # --------------------------------------------------
    # 6. 박스오피스 결과 확인
    # --------------------------------------------------

    boxoffice_result = result.get("boxOfficeResult")

    if not boxoffice_result:
        return {
            "success": False,
            "message": (
                "KOBIS에서 박스오피스 결과를 찾을 수 없습니다.\n\n"
                "API 응답이나 조회 날짜를 확인해 주세요."
            ),
            "data": None
        }


    # 영화 목록 가져오기
    movie_list = boxoffice_result.get(
        "dailyBoxOfficeList",
        []
    )


    # 영화 목록이 비어 있는 경우
    if not movie_list:
        return {
            "success": False,
            "message": (
                f"{display_date}의 영화 목록이 비어 있습니다.\n\n"
                "다음 사항을 확인해 주세요.\n\n"
                "1. KOBIS에 해당 날짜의 박스오피스 자료가 등록되었는지\n"
                "2. KOBIS API가 정상적으로 작동하는지\n"
                "3. KOBIS_KEY가 정상적으로 등록되어 있는지"
            ),
            "data": None
        }


    # 정상적으로 데이터를 가져온 경우
    return {
        "success": True,
        "message": "",
        "data": movie_list
    }


# --------------------------------------------------
# 7. API 실행
# --------------------------------------------------

result = get_boxoffice(target_date)


# --------------------------------------------------
# 8. API 요청 실패 시 안내
# --------------------------------------------------

if not result["success"]:

    st.error(result["message"])

    st.info(
        "💡 Streamlit Cloud에서 문제가 발생했다면 "
        "Settings → Secrets에 다음과 같이 등록되어 있는지 확인하세요.\n\n"
        "KOBIS_KEY = \"발급받은_인증키\""
    )

    st.stop()


# --------------------------------------------------
# 9. 데이터를 데이터프레임으로 변환
# --------------------------------------------------

movies = result["data"]

df = pd.DataFrame(movies)


# --------------------------------------------------
# 10. 숫자 데이터를 실제 숫자로 변환
# --------------------------------------------------
# KOBIS API에서는 숫자도 문자열로 전달됩니다.
# 그래프나 정렬에 사용하려면 숫자형으로 변환해야 합니다.

number_columns = [
    "rank",
    "rankInten",
    "audiCnt",
    "audiAcc",
    "scrnCnt",
    "showCnt"
]

for column in number_columns:

    if column in df.columns:

        # "-" 같은 값도 있을 수 있으므로
        # 숫자로 바꿀 수 없는 값은 0으로 처리합니다.
        df[column] = pd.to_numeric(
            df[column],
            errors="coerce"
        ).fillna(0).astype(int)


# --------------------------------------------------
# 11. 순위 변동 표시용 함수
# --------------------------------------------------
# 실제 계산에는 숫자를 사용하고,
# 화면에 표시할 때만 ▲ / ▼ 기호를 붙입니다.

def format_rank_change(value):

    if value > 0:
        return f"▲ {value}"

    elif value < 0:
        return f"▼ {abs(value)}"

    else:
        return "-"


# 순위 변동 표시용 컬럼을 새로 만듭니다.
df["rankChange"] = df["rankInten"].apply(
    format_rank_change
)


# --------------------------------------------------
# 12. 조회 날짜 표시
# --------------------------------------------------

st.subheader(f"📅 {display_date} 박스오피스")

st.write(
    f"전날인 **{display_date}**에 집계된 영화관 박스오피스입니다."
)


# --------------------------------------------------
# 13. 1위 영화 정보
# --------------------------------------------------

if len(df) > 0:

    first_movie = df.iloc[0]

    st.subheader("🏆 오늘의 1위")

    # 영화 제목
    st.markdown(
        f"## {first_movie['movieNm']}"
    )

    # 순위 변동 표시
    rank_change = first_movie["rankChange"]

    if first_movie["rankInten"] > 0:
        st.success(f"순위 변동: {rank_change}")

    elif first_movie["rankInten"] < 0:
        st.error(f"순위 변동: {rank_change}")

    else:
        st.info("순위 변동: -")


    # 3개의 지표 카드를 나란히 표시
    card1, card2, card3 = st.columns(3)


    # 당일 관객수
    with card1:

        st.metric(
            "당일 관객수",
            f"{first_movie['audiCnt']:,}명"
        )


    # 누적 관객수
    with card2:

        st.metric(
            "누적 관객수",
            f"{first_movie['audiAcc']:,}명"
        )


    # 스크린수
    with card3:

        st.metric(
            "스크린수",
            f"{first_movie['scrnCnt']:,}개"
        )


# --------------------------------------------------
# 14. 관객수 상위 5편 그래프
# --------------------------------------------------

st.subheader("📊 관객수 상위 5편")


# 관객수 기준으로 내림차순 정렬
top5 = (
    df.sort_values(
        by="audiCnt",
        ascending=False
    )
    .head(5)
    .copy()
)


# 영화명과 관객수만 그래프에 사용
top5_chart = top5[
    ["movieNm", "audiCnt"]
].set_index("movieNm")


# 막대그래프
st.bar_chart(
    top5_chart,
    y="audiCnt"
)

st.caption(
    "※ 당일 관객수 기준 상위 5편입니다."
)


# --------------------------------------------------
# 15. 전체 박스오피스 표
# --------------------------------------------------

st.subheader("🎞️ 전체 박스오피스")


# 표에 필요한 데이터만 선택
table_df = df[
    [
        "rank",
        "rankChange",
        "movieNm",
        "openDt",
        "audiCnt",
        "audiAcc",
        "scrnCnt"
    ]
].copy()


# 컬럼명을 한국어로 변경
table_df.columns = [
    "순위",
    "순위 변동",
    "영화명",
    "개봉일",
    "관객수",
    "누적관객",
    "스크린수"
]


# --------------------------------------------------
# 16. 숫자에 천 단위 쉼표 표시
# --------------------------------------------------
# 표에서는 숫자를 읽기 편하게 표시합니다.

for column in [
    "순위",
    "관객수",
    "누적관객",
    "스크린수"
]:

    table_df[column] = table_df[column].apply(
        lambda value: f"{value:,}"
    )


# --------------------------------------------------
# 17. 최종 표 출력
# --------------------------------------------------

st.dataframe(
    table_df,
    use_container_width=True,
    hide_index=True
)


# --------------------------------------------------
# 18. 데이터 출처
# --------------------------------------------------

st.caption(
    "데이터 출처: 영화관입장권통합전산망(KOBIS) "
    "일별 박스오피스 Open API"
)
```
