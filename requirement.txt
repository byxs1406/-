import streamlit as st
import datetime
import pytz

def get_time_in_timezone(timezone_str):
    """
    指定されたタイムゾーンの現在時刻を取得し、整形して返します。
    """
    try:
        tz = pytz.timezone(timezone_str)
        now = datetime.datetime.now(tz)
        return now
    except pytz.UnknownTimeZoneError:
        return None

def format_timedelta(td):
    """
    timedeltaオブジェクトを「X時間 Y分」の形式に整形します。
    """
    total_seconds = int(td.total_seconds())
    hours, remainder = divmod(total_seconds, 3600)
    minutes, _ = divmod(remainder, 60)
    
    sign = ""
    if hours > 0:
        sign = "+"
    elif hours < 0:
        sign = "" # -記号はdivmodが自動的に付与
    
    return f"{sign}{hours}時間 {abs(minutes)}分"

st.set_page_config(layout="wide", page_title="世界時計")
st.title("🌎 世界時計")

# --- セッションステートの初期化 ---
if 'cities' not in st.session_state:
    st.session_state.cities = {
        "東京": "Asia/Tokyo",
        "ニューヨーク": "America/New_York",
        "ロンドン": "Europe/London",
        "パリ": "Europe/Paris",
        "ドバイ": "Asia/Dubai",
        "シドニー": "Australia/Sydney",
        "シンガポール": "Asia/Singapore",
        "ベルリン": "Europe/Berlin",
        "リオデジャネイロ": "America/Sao_Paulo",
        "北京": "Asia/Shanghai",
    }

# --- 主要都市の現在時刻表示 ---
st.header("主要都市の現在時刻")

num_cols = 3
cols = st.columns(num_cols)
col_index = 0

for city_name, timezone_str in list(st.session_state.cities.items()):
    with cols[col_index]:
        st.subheader(city_name)
        current_time = get_time_in_timezone(timezone_str)
        if current_time:
            st.info(current_time.strftime("%Y年%m月%d日 %H時%M分%S秒 %Z%z"))
        else:
            st.error("時刻取得エラー")
    col_index = (col_index + 1) % num_cols

st.markdown("---")

# --- 都市追加機能 ---
st.header("新しい都市を追加")

with st.expander("都市を追加するフォームを開く"):
    new_city_name = st.text_input("追加したい都市名（例: ロサンゼルス）")
    new_timezone = st.text_input(
        "タイムゾーン（例: America/Los_Angeles, Asia/Shanghai）",
        help="有効なタイムゾーンは [IANA Time Zone Database](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones) で検索できます。"
    )
    
    if st.button("都市を追加する"):
        if new_city_name and new_timezone:
            if new_timezone in pytz.all_timezones:
                if new_city_name not in st.session_state.cities:
                    st.session_state.cities[new_city_name] = new_timezone
                    st.success(f"'{new_city_name}' を世界時計に追加しました！")
                    st.experimental_rerun()
                else:
                    st.warning(f"'{new_city_name}' は既にリストに存在します。")
            else:
                st.error("入力されたタイムゾーンは無効です。正しいタイムゾーンを入力してください。")
        else:
            st.warning("都市名とタイムゾーンの両方を入力してください。")

st.markdown("---")

# --- 都市削除機能 (オプション) ---
st.header("都市を削除")
with st.expander("都市を削除するフォームを開く"):
    cities_to_delete = st.multiselect(
        "削除したい都市を選択してください:",
        options=list(st.session_state.cities.keys()),
        key="delete_city_multiselect" # キーを追加して識別
    )
    if st.button("選択した都市を削除", key="delete_city_button"): # キーを追加して識別
        if cities_to_delete:
            for city in cities_to_delete:
                if city in st.session_state.cities:
                    del st.session_state.cities[city]
                    st.success(f"'{city}' を削除しました。")
            st.experimental_rerun()
        else:
            st.info("削除する都市を選択してください。")

st.markdown("---")

## --- 時差計算機能 ---
st.header("📅 都市間の時差を計算")

all_city_names = list(st.session_state.cities.keys())

if len(all_city_names) < 2:
    st.warning("時差を計算するには、最低2つの都市が必要です。")
else:
    col1, col2 = st.columns(2)
    
    with col1:
        city1_name = st.selectbox(
            "最初の都市を選択:",
            options=all_city_names,
            index=0, # デフォルトで最初の都市を選択
            key="diff_city1"
        )
    
    with col2:
        # 最初の都市とは異なる都市を選択肢にする
        available_cities_for_city2 = [city for city in all_city_names if city != city1_name]
        
        # available_cities_for_city2が空でないことを確認
        if not available_cities_for_city2:
            st.warning("選択できる2つ目の都市がありません。")
        else:
            # 2番目の都市の初期インデックスを設定 (利用可能な都市の最初のもの)
            initial_index_city2 = 0
            city2_name = st.selectbox(
                "2番目の都市を選択:",
                options=available_cities_for_city2,
                index=initial_index_city2, # デフォルトで最初の利用可能な都市を選択
                key="diff_city2"
            )

    if city1_name and city2_name and city1_name != city2_name:
        timezone_str1 = st.session_state.cities[city1_name]
        timezone_str2 = st.session_state.cities[city2_name]
        
        time1 = get_time_in_timezone(timezone_str1)
        time2 = get_time_in_timezone(timezone_str2)
        
        if time1 and time2:
            # 時差を計算 (time2 - time1)
            time_difference = time2.replace(microsecond=0) - time1.replace(microsecond=0)
            
            # 結果を分かりやすく表示
            st.success(f"**{city2_name}** は **{city1_name}** より")
            st.markdown(f"### **{format_timedelta(time_difference)}**")
            st.success(f"進んでいます。")
            
            # 逆方向の時差も表示（オプション）
            st.caption(f"（言い換えると、{city1_name} は {city2_name} より {format_timedelta(-time_difference)} 遅れています。）")
        else:
            st.error("選択された都市のタイムゾーン情報に問題があります。")
    elif city1_name == city2_name and len(all_city_names) >= 2:
        st.info("異なる都市を選択してください。")


st.markdown("---")
st.caption("この世界時計アプリは Streamlit と pytz ライブラリを使用して作成されました。")

# アプリケーションの実行方法:
# 1. Pythonとpipがインストールされていることを確認します。
# 2. 必要なライブラリをインストールします: `pip install streamlit pytz`
# 3. このコードを `world_clock_app.py` などのファイル名で保存します。
# 4. ターミナルで `streamlit run world_clock_app.py` を実行します。
