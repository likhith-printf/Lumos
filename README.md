# Lumos
Lumos is a custom Python and Streamlit web application that eliminates late-night campus isolation by dynamically grouping solo night-walkers into destination-based "Walking Trains" while leveraging a Gemini AI keyword-interceptor to instantly match exhausted students with nearby peers for coffee breaks.
import streamlit as st
import pandas as pd
import pydeck as pdk
from google import genai

# --- INITIALIZE GEMINI CLIENT ---
client = genai.Client(api_key="YOUR_API_KEY_HERE")

st.set_page_config(page_title="Lumos Dashboard", layout="wide")

# --- CUSTOM CSS FOR DARK MODE & ACTION BUTTON ---
st.markdown(
    """
    <style>
    /* Dark Theme Core Background */
    .stApp {
        background-color: #0E1117;
        color: #FAFAFA;
    }
    /* Sleek Dropdown styling visibility fixes */
    .stSelectbox div div {
        background-color: #1E222B !important;
        color: #FAFAFA !important;
    }
    /* Glowing Action Button */
    div.stButton > button:first-child {
        background-color: #0096FF !important;
        color: #FFFFFF !important;
        border: 2px solid #00D2FF !important;
        font-weight: bold !important;
        font-size: 16px !important;
        border-radius: 8px !important;
        padding: 10px 24px !important;
        transition: all 0.3s ease;
        box-shadow: 0px 0px 12px rgba(0, 150, 255, 0.6);
        width: 100%;
    }
    div.stButton > button:first-child:hover {
        background-color: #00D2FF !important;
        box-shadow: 0px 0px 20px rgba(0, 210, 255, 0.9);
        color: #0E1117 !important;
    }
    </style>
    """,
    unsafe_allow_html=True
)

st.title("💡 Lumos - Campus Night Safety Engine")

# --- INITIALIZE MEMORY STATE ---
if "passenger_count" not in st.session_state:
    st.session_state.passenger_count = 3
if "messages" not in st.session_state:
    st.session_state.messages = [{"role": "assistant", "content": "Hoo... 🦉 Pulling an all-nighter at VIT? How are your energy levels holding up?"}]

# --- CREATE SCREEN COLUMNS ---
col1, col2 = st.columns(2)

# ==========================================================
# 🎨 LEFT SIDE: PYDECK OPEN-TERRAIN MAP WITH LINES
# ==========================================================
with col1:
    st.header("📍 Campus Energy Map")
    st.caption("Real-time localized routing path configuration grid")
    
    # 1. EXPANDED CAMPUS LANDMARK DATABASE WITH YOUR EXACT COORDINATES
    locations_db = {
        "Main Gate": {"lat": 12.9683384, "lon": 79.1551231, "short": "Main Gate"},
        "SJT (Silver Jubilee Tower)": {"lat": 12.9712, "lon": 79.1639, "short": "SJT Tower"},
        "TT (Technology Tower)": {"lat": 12.9711, "lon": 79.1638, "short": "TT Tower"},
        "Main Block": {"lat": 12.9705, "lon": 79.1593, "short": "Main Block"},
        "A Block": {"lat": 12.9723287, "lon": 79.156012, "short": "Block A"},
        "B Block": {"lat": 12.9732112, "lon": 79.1566905, "short": "Block B"},
        "C Block": {"lat": 12.9732112, "lon": 79.1566905, "short": "Block C"},
        "D Block": {"lat": 12.9728769, "lon": 79.1596243, "short": "Block D"},
        "E Block": {"lat": 12.9723305, "lon": 79.1588424, "short": "Block E"},
        "F Block": {"lat": 12.9732112, "lon": 79.1566905, "short": "Block F"},
        "G Block": {"lat": 12.9731505, "lon": 79.159538, "short": "Block G"},
        "H Block": {"lat": 12.9723287, "lon": 79.156012, "short": "Block H"},
        "J Block": {"lat": 12.9719412, "lon": 79.1564212, "short": "Block J"},
        "K Block": {"lat": 12.972615, "lon": 79.1617686, "short": "Block K"},
        "L Block": {"lat": 12.972615, "lon": 79.1617686, "short": "Block L"},
        "M Block": {"lat": 12.9717709, "lon": 79.1595553, "short": "Block M"},
        "N Block": {"lat": 12.9744253, "lon": 79.1642498, "short": "Block N"},
        "P Block": {"lat": 12.9744268, "lon": 79.1642487, "short": "Block P"},
        "Q Block": {"lat": 12.9717709, "lon": 79.1595553, "short": "Block Q"},
        "R Block": {"lat": 12.972292, "lon": 79.1616549, "short": "Block R"},
        "S Block": {"lat": 12.9717709, "lon": 79.159072, "short": "Block S"},
        "T Block": {"lat": 12.9744482, "lon": 79.1642623, "short": "Block T"},
        "LA Block": {"lat": 12.9684379, "lon": 79.1578135, "short": "LA Block"},
        "LC Block": {"lat": 12.9710083, "lon": 79.1598333, "short": "LC Block"},
        "LD Block": {"lat": 12.9710973, "lon": 79.1606593, "short": "LD Block"},
        "LE Block": {"lat": 12.9710642, "lon": 79.1593961, "short": "LE Block"},
        "LF Block": {"lat": 12.9711234, "lon": 79.1623896, "short": "LF Block"},
        "LG Block": {"lat": 12.968348, "lon": 79.1587924, "short": "LG Block"}
    }
    
    # 2. Interactive Dropdown Selection Menus
    st.subheader("⚙️ Route Configuration")
    col_from, col_to = st.columns(2)
    
    with col_from:
        start_loc = st.selectbox("Departure Hub", list(locations_db.keys()), index=1) # Defaults to SJT
    with col_to:
        end_loc = st.selectbox("Destination Hub", list(locations_db.keys()), index=14) # Defaults to L Block
        
    start_lat = locations_db[start_loc]["lat"]
    start_lon = locations_db[start_loc]["lon"]
    end_lat = locations_db[end_loc]["lat"]
    end_lon = locations_db[end_loc]["lon"]

    # 3. Construct Datastructures for Mapping Layer Rendering
    points_data = pd.DataFrame([
        {"lat": start_lat, "lon": start_lon, "name": locations_db[start_loc]["short"]},
        {"lat": end_lat, "lon": end_lon, "name": locations_db[end_loc]["short"]}
    ])
    
    line_data = pd.DataFrame([{"start": [start_lon, start_lat], "end": [end_lon, end_lat]}])

    # Red dot destination nodes
    hub_layer = pdk.Layer(
        "ScatterplotLayer",
        data=points_data,
        get_position="[lon, lat]",
        get_color="[255, 75, 75, 200]",
        get_radius=20,
        pickable=True
    )
    
    # Restored Blue Path Line
    route_layer = pdk.Layer(
        "LineLayer",
        data=line_data,
        get_source_position="start",
        get_target_position="end",
        get_color="[0, 150, 255, 255]",
        get_width=6
    )
    
    # Text labels over nodes
    text_layer = pdk.Layer(
        "TextLayer",
        data=points_data,
        get_position="[lon, lat]",
        get_text="name",
        get_size=18,
        get_color="[0, 0, 0, 255]", # High contrast black font for light terrain tiles
        get_alignment_baseline="'bottom'",
        get_pixel_offset="[0, -15]"
    )
    
    # Using open-source CartoDB light map tiles to ensure roads show up with zero key errors
    st.pydeck_chart(pdk.Deck(
        map_style="https://cartocdn.com",
        initial_view_state=pdk.ViewState(
            latitude=(start_lat + end_lat) / 2,
            longitude=(start_lon + end_lon) / 2,
            zoom=15,
            pitch=0
        ),
        layers=[hub_layer, route_layer, text_layer]
    ))
    
    st.markdown("---")
    
    # 4. Automated Walking Trains Output Dashboard
    st.header("🚆 Automated Walking Trains")
    
    if start_loc == end_loc:
        st.warning("⚠️ Please select two different hubs to generate a walking train route.")
    else:
        st.info(f"🚀 **{start_loc} ➔ {end_loc}** | Leaving in **5 mins** | {st.session_state.passenger_count} night-walkers joined.")
        if st.button("🙋‍♂️ Tap to Join Train"):
            st.session_state.passenger_count += 1
            st.success(f"Added! Meet at the {start_loc} Ground Floor entrance.")
            st.rerun()

# ==========================================================
# 🧠 RIGHT SIDE: TEAM MEMBER 2 & 3 (AI Chat Engine)
# ==========================================================
with col2:
    st.header("🦉 Night Owl Check-In")
    st.caption("AI Companion for Mental Fatigue & Break Matching")
    
    for msg in st.session_state.messages:
        with st.chat_message(msg["role"]):
            st.write(msg["content"])
            
    if user_input := st.chat_input("Tell the Owl how you are feeling..."):
        st.session_state.messages.append({"role": "user", "content": user_input})
        with st.chat_message("user"):
            st.write(user_input)
            
        with st.spinner("Owl is thinking..."):
            try:
                response = client.models.generate_content(
                    model='gemini-2.5-flash',
                    contents=user_input,
                    config={
                        'system_instruction': "You are 'Night Owl', a friendly, comforting campus AI assistant for VIT Vellore students pulling all-nighters. Keep responses concise (under 2 sentences), warm, and focused on checking their mental exhaustion."
                    }
                )
                ai_reply = response.text
            except Exception as e:
                ai_reply = "Hoo... 🦉 My connection to the safety network is blinking, but I'm still here tracking the campus grid with you!"
                
        if any(word in user_input.lower() for word in ["tired", "sleepy", "exhausted", "coffee", "tea"]):
            ai_reply += f"\n\n🚨 **Lumos Match!** Another student near **{start_loc}** is also feeling fatigued. Go grab a 10-minute tea break together!"
            
        st.session_state.messages.append({"role": "assistant", "content": ai_reply})
        st.rerun()




