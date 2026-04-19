# Railway-proactive-alert-index

## Step 1: Install dab-py
```python
1.   !pip install --upgrade dab-py
 
## Step 2: Run the main code

```python
from dabpy import HISCentralClient, Constraints
import pandas as pd
from ipyleaflet import Map, basemaps, Rectangle, DrawControl, Marker, AwesomeIcon
import matplotlib.pyplot as plt
import matplotlib.dates as mdates
import ipywidgets as widgets
from ipywidgets import Layout
import numpy as np
from IPython.display import display, clear_output
from google.colab import userdata, output

# CRITICAL FOR COLAB: Enable custom interactive widgets for the map drawing tool
output.enable_custom_widget_manager()

# ==========================================
# 1. HYBRID CONFIGURATION (PREDEFINED + CUSTOM)
# ==========================================
SITES = {
    "Custom Search": {
        "mode": "dynamic_bbox",
        "bbox": {"south": 45.480, "north": 45.680, "west": 9.120, "east": 9.250},
        "start": "2014-11-14T00:00:00Z", "end": "2014-11-16T00:00:00Z",
        "infra_threshold": 4.5, "warning_msg": "Est. Impact in",
        "unit_lvl": "m", "unit_rain": "mm",
        "grad_orange": 0.20, "grad_red": 0.50
    },
    "Genoa (Nov 2011 Flood)": {
        "mode": "hardcoded_id",
        "bbox": {"south": 44.380, "north": 44.450, "west": 8.900, "east": 8.980},
        "level": {
            "id": "BD3F457C94C7EABD4B4486C50E610ECF145093C9", "lat": 44.41094, "lon": 8.95395, "name": "Firpo Station",
            "conv": 1.0, "kw": None, "obs_prop": "water level", "index": None
        },
        "precipitation": {
            "id": "4B1A2CD9FDE76C2CAD6EC894DA82B6D9E31D8D28", "lat": 44.41500, "lon": 8.95000, "name": "Genoa Rain Gauge",
            "conv": 1.0, "kw": None, "obs_prop": "precipitation",
            "index": 0,
            "is_cumulative": False
        },
        "start": "2011-11-04T06:00:00Z", "end": "2011-11-04T20:00:00Z",
        "infra_threshold": 4.5, "warning_msg": "Est. Track Submersion in",
        "unit_lvl": "m", "unit_rain": "mm",
        "grad_orange": 0.20, "grad_red": 0.50
    },
    "Milan (Seveso Flood 2014)": {
        "mode": "hardcoded_id",
        "bbox": {"south": 45.480, "north": 45.680, "west": 9.120, "east": 9.250},
        "level": {
            "id": "3D56FCDD48E0EE8AA187C1F58DDCA4BEDB9B125C", "lat": 45.518, "lon": 9.189, "name": "Milano Niguarda",
            "conv": 1.0, "kw": None, "obs_prop": "level",
            "index": 1
        },
        "precipitation": {
            "id": "1099207D109FEFEF10BA5991165B41758906732E", "lat": 45.520, "lon": 9.200, "name": "Cinisello Parco Nord",
            "conv": 1.0, "kw": None, "obs_prop": "precipitation",
            "index": 3,
            "is_cumulative": False
        },
        "start": "2014-11-14T12:00:00Z", "end": "2014-11-16T23:59:00Z",
        "infra_threshold": 450.0, "warning_msg": "Est. Culvert Overtopping in",
        "unit_lvl": "cm", "unit_rain": "mm",
        "grad_orange": 0.10, "grad_red": 0.25
    }
}

active_df_lvl = pd.DataFrame()
active_df_rain = pd.DataFrame()
active_lvl_name = ""
active_rain_name = ""

# ==========================================
# 2. UI WIDGETS
# ==========================================
header = widgets.HTML("<h2 style='color: #2c3e50; border-bottom: 2px solid #ecf0f1; padding-bottom: 10px;'>🚉 Spatial Infrastructure Risk Dashboard</h2>")
scenario_dropdown = widgets.Dropdown(options=list(SITES.keys()), value="Milan (Seveso Flood 2014)", description='Scenario:', layout=Layout(width='95%'))

style = {'description_width': 'initial'}
w_south = widgets.FloatText(description='South:', layout=Layout(width='45%'), style=style)
w_north = widgets.FloatText(description='North:', layout=Layout(width='45%'), style=style)
w_west = widgets.FloatText(description='West:', layout=Layout(width='45%'), style=style)
w_east = widgets.FloatText(description='East:', layout=Layout(width='45%'), style=style)
bbox_box = widgets.VBox([
    widgets.HTML("<b>Bounding Box (WGS84):</b> <i style='font-size: 11px;'>(Draw on map to update)</i>"),
    widgets.HBox([w_north], layout=Layout(justify_content='center')),
    widgets.HBox([w_west, w_east], layout=Layout(justify_content='space-between')),
    widgets.HBox([w_south], layout=Layout(justify_content='center'))
], layout=Layout(align_items='stretch', margin='10px 0', padding='10px', border='1px solid #bdc3c7', border_radius='5px'))

w_start = widgets.Text(description='Start (UTC):', layout=Layout(width='95%'), style=style)
w_end = widgets.Text(description='End (UTC):', layout=Layout(width='95%'), style=style)
w_thresh = widgets.FloatText(description='Threshold:', layout=Layout(width='95%'), style=style)

btn_analyze = widgets.Button(description='Fetch Area & Analyze', button_style='primary', icon='rocket', layout=Layout(width='95%', margin='10px 0', height='40px'))

map_output = widgets.Output(layout=Layout(width='100%', height='360px', border='1px solid #ecf0f1', border_radius='8px'))
kpi_output = widgets.Output(layout=Layout(width='100%'))
chart_output = widgets.Output(layout=Layout(width='100%'))
slider_output = widgets.Output(layout=Layout(width='100%'))
time_display = widgets.HTML("<div style='background: #34495e; color: white; padding: 10px 15px; border-radius: 5px; font-weight: bold; margin: 10px 0; font-size: 16px;'>🕒 Simulation Time: --</div>")

# Slider declared globally so observers can access it
play = widgets.Play(value=0, min=0, max=100, step=1, interval=600)
slider = widgets.IntSlider(min=0, max=100, step=1, value=0, description='Timeline:', layout=Layout(width='85%'))
widgets.jslink((play, 'value'), (slider, 'value'))

# ==========================================
# 3. INTERACTIVE MAP INITIALIZATION
# ==========================================
def draw_base_map(change=None):
    scenario = SITES[scenario_dropdown.value]

    if change:
        w_south.value, w_north.value = scenario['bbox']['south'], scenario['bbox']['north']
        w_west.value, w_east.value = scenario['bbox']['west'], scenario['bbox']['east']
        w_start.value, w_end.value = scenario['start'], scenario['end']
        w_thresh.value = scenario['infra_threshold']

    with map_output:
        clear_output(wait=True)
        c_lat = (w_south.value + w_north.value) / 2
        c_lon = (w_west.value + w_east.value) / 2

        m = Map(center=(c_lat, c_lon), zoom=10, basemap=basemaps.CartoDB.Positron)
        draw_control = DrawControl(rectangle={"shapeOptions": {"color": "#e74c3c", "fillOpacity": 0.1}})
        draw_control.polyline = {}; draw_control.polygon = {}; draw_control.circlemarker = {}; draw_control.marker = {}

        def handle_draw(target, action, geo_json):
            if action == 'created' and geo_json['geometry']['type'] == 'Polygon':
                coords = geo_json['geometry']['coordinates'][0]
                lons = [c[0] for c in coords]; lats = [c[1] for c in coords]
                w_south.value, w_north.value = round(min(lats), 4), round(max(lats), 4)
                w_west.value, w_east.value = round(min(lons), 4), round(max(lons), 4)

        draw_control.on_draw(handle_draw)
        m.add_control(draw_control)
        m.add_layer(Rectangle(bounds=((w_south.value, w_west.value), (w_north.value, w_east.value)), color='#3498db', fill_color='#3498db', fill_opacity=0.1))

        icon_lvl = AwesomeIcon(name='tint', marker_color='white', icon_color='darkblue', spin=False)
        icon_rain = AwesomeIcon(name='cloud', marker_color='white', icon_color='cadetblue', spin=False)

        if scenario['mode'] == 'hardcoded_id':
            m.add_layer(Marker(location=(scenario['level']['lat'], scenario['level']['lon']), title=f"Level: {scenario['level']['name']}", icon=icon_lvl))
            m.add_layer(Marker(location=(scenario['precipitation']['lat'], scenario['precipitation']['lon']), title=f"Rain: {scenario['precipitation']['name']}", icon=icon_rain))
        display(m)

scenario_dropdown.observe(draw_base_map, names='value')

def get_level_alert(grad, unit='m', t_orange=0.20, t_red=0.50):
    val = grad / 100.0 if unit == 'cm' else grad
    if val >= t_red: return '#e74c3c', 'RED (CRITICAL)'
    elif val >= t_orange: return '#e67e22', 'ORANGE (HIGH RISK)'
    elif val < 0: return '#3498db', 'BLUE (RECEDING)'
    return '#2ecc71', 'GREEN (NORMAL)'

def get_rain_alert(rain):
    if rain >= 50.0: return '#e74c3c', 'RED (EXTREME)'
    elif rain >= 30.0: return '#e67e22', 'ORANGE (HIGH RISK)'
    elif rain >= 10.0: return '#f1c40f', 'YELLOW (MODERATE)'
    return '#3498db', 'BLUE (NORMAL)'

# ==========================================
# 4. DATA FETCHING LOGIC
# ==========================================
def process_data(df, target_col, start, end):
    full_time = pd.date_range(start=pd.to_datetime(start), end=pd.to_datetime(end), freq='1H', tz='UTC')

    if target_col == 'Level':
        df = df.resample('1H').mean(numeric_only=True).reindex(full_time)
        df['Level'] = df['Level'].ffill().bfill()
        df['Gradient_1h'] = (df['Level'] - df['Level'].shift(1)).fillna(0)
    else:
        df = df.resample('1H').max(numeric_only=True).reindex(full_time).fillna(0)
        if df['Precipitation'].is_monotonic_increasing and df['Precipitation'].max() > 0:
            df['Intensity_1h'] = df['Precipitation'].diff().fillna(0).clip(lower=0)
        else:
            df['Intensity_1h'] = df['Precipitation']

    return df

def fetch_by_id(feature_id, obs_prop, start, end, keyword, conversion, index=None):
    try:
        token = 'his_central-568a4888-d6bd-4be3-b7a6-d9887997bb0f'
        client = HISCentralClient(token=token, view="his-central")
        c = Constraints(feature=feature_id, observedProperty=obs_prop, ontology="his-central", limit="100")
        obs_list = client.get_observations(c)
        if not obs_list: return pd.DataFrame()

        if index is not None and len(obs_list) > index:
            selected_id = obs_list[index].id
        else:
            selected_id = obs_list[0].id
            if keyword:
                for obs in obs_list:
                    obs_text = str(vars(obs)).lower() + getattr(obs, 'title', '').lower()
                    if keyword.lower() in obs_text:
                        selected_id = obs.id
                        break

        data = client.get_observation_with_data(selected_id, begin=start, end=end)
        df = client.points_to_df(data)
        if df.empty: return pd.DataFrame()

        df['Time'] = pd.to_datetime(df['Time'], utc=True)
        df.set_index('Time', inplace=True)
        df = df[~df.index.duplicated(keep='last')]

        col_name = 'Level' if 'level' in obs_prop.lower() else 'Precipitation'
        df.rename(columns={'Value': col_name}, inplace=True)
        df[col_name] = pd.to_numeric(df[col_name], errors='coerce') * conversion

        return process_data(df, col_name, start, end)
    except Exception as e:
        print(f"ID Fetch Error: {e}")
        return pd.DataFrame()

def fetch_by_bbox(bbox_list, obs_prop, start, end):
    try:
        token = userdata.get('token-his-central')
        client = HISCentralClient(token=token, view="his-central")
        c = Constraints(bbox=bbox_list, observedProperty=obs_prop, ontology="his-central", limit="50")
        obs_list = client.get_observations(c)
        if not obs_list: return pd.DataFrame(), "No Sensors Found", 0, 0

        for obs in obs_list:
            st_name = getattr(obs, 'featureOfInterest', None)
            st_name = st_name.name if st_name and hasattr(st_name, 'name') else "Unknown Station"
            lat = obs.featureOfInterest.shape[0] if st_name != "Unknown Station" else 0
            lon = obs.featureOfInterest.shape[1] if st_name != "Unknown Station" else 0

            try:
                data = client.get_observation_with_data(obs.id, begin=start, end=end)
                df = client.points_to_df(data)
                if df.empty: continue

                df['Time'] = pd.to_datetime(df['Time'], utc=True)
                df.set_index('Time', inplace=True)
                df = df[~df.index.duplicated(keep='last')]
                col_name = 'Level' if 'level' in obs_prop.lower() else 'Precipitation'
                df.rename(columns={'Value': col_name}, inplace=True)
                df[col_name] = pd.to_numeric(df[col_name], errors='coerce')

                df = process_data(df, col_name, start, end)

                if 'level' in obs_prop.lower() and df['Level'].max() > 100:
                    df['Level'] = df['Level'] * 0.01
                    df['Gradient_1h'] = df['Gradient_1h'] * 0.01

                return df, st_name, lat, lon
            except: continue
        return pd.DataFrame(), "No valid historical data found in area", 0, 0
    except Exception as e:
        print(f"BBox Fetch Error: {e}")
        return pd.DataFrame(), "Error", 0, 0

# ==========================================
# 5. RENDER ENGINE (DYNAMIC THRESHOLD BINDS)
# ==========================================
def render_frame(change=None):
    if active_df_lvl.empty or active_df_rain.empty: return

    step = slider.value
    c_df_lvl = active_df_lvl.iloc[:step+1]
    c_df_rain = active_df_rain.iloc[:step+1]
    current_time = c_df_lvl.index[-1].strftime('%d %b %Y - %H:%M UTC')

    time_display.value = f"<div style='background: #34495e; color: white; padding: 10px 15px; border-radius: 5px; font-weight: bold; margin: 10px 0; font-size: 16px;'>🕒 Simulation Time: {current_time}</div>"

    scenario = SITES[scenario_dropdown.value]
    u_lvl = scenario.get('unit_lvl', 'm')
    u_rain = scenario.get('unit_rain', 'mm')
    t_org = scenario.get('grad_orange', 0.20)
    t_red = scenario.get('grad_red', 0.50)

    thresh = w_thresh.value

    with kpi_output:
        clear_output(wait=True)
        curr_lvl = c_df_lvl['Level'].iloc[-1]
        curr_grad = c_df_lvl['Gradient_1h'].iloc[-1]
        c_lvl, t_lvl = get_level_alert(curr_grad, unit=u_lvl, t_orange=t_org, t_red=t_red)

        pred = ""
        if curr_lvl >= thresh:
            pred = f"<div style='margin-top: 10px; background-color: #c0392b; padding: 8px; border-radius: 5px; color: white; font-weight: bold; text-align: center;'>🚨 INFRASTRUCTURE BREACHED</div>"
        elif curr_grad > 0:
            hrs = (thresh - curr_lvl) / curr_grad
            if hrs < 12: pred = f"<div style='margin-top: 10px; background-color: #fff3cd; padding: 8px; border-radius: 5px; border: 1px solid #ffeeba; color: #856404; font-weight: bold; text-align: center;'>⚠️ Est. Submersion ({thresh} {u_lvl}) in:<br><span style='font-size: 18px;'>{hrs:.1f} hours</span></div>"

        lvl_card = f"""<div style="background-color: white; border-left: 6px solid {c_lvl}; padding: 15px; border-radius: 8px; box-shadow: 0 2px 4px rgba(0,0,0,0.1); margin-bottom: 15px;">
            <h4 style="margin: 0; color: #7f8c8d; font-size: 12px;">{active_lvl_name.upper()}</h4>
            <h1 style="margin: 5px 0; color: #2c3e50; font-size: 28px;">{curr_grad:.2f} {u_lvl}/h</h1>
            <p style="margin: 0 0 10px 0; color: #7f8c8d; font-size: 14px;">Abs Level: {curr_lvl:.2f} {u_lvl}</p>
            <div style="background-color: {c_lvl}; color: white; padding: 4px 8px; border-radius: 15px; display: inline-block; font-size: 12px; font-weight: bold;">{t_lvl}</div>
            {pred}
        </div>"""

        curr_rain_int = c_df_rain['Intensity_1h'].iloc[-1]
        c_rn, t_rn = get_rain_alert(curr_rain_int)

        rain_card = f"""<div style="background-color: white; border-left: 6px solid {c_rn}; padding: 15px; border-radius: 8px; box-shadow: 0 2px 4px rgba(0,0,0,0.1);">
            <h4 style="margin: 0; color: #7f8c8d; font-size: 12px;">{active_rain_name.upper()}</h4>
            <h1 style="margin: 5px 0; color: #2c3e50; font-size: 28px;">{curr_rain_int:.1f} {u_rain}/h</h1>
            <div style="background-color: {c_rn}; color: white; padding: 4px 8px; border-radius: 15px; display: inline-block; font-size: 12px; font-weight: bold; margin-top: 5px;">{t_rn}</div>
        </div>"""
        display(widgets.HTML(lvl_card + rain_card))

    with chart_output:
        clear_output(wait=True)
        plt.style.use('seaborn-v0_8-whitegrid')

        fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(15, 6))

        ax1.set_title(f"Water Level: {active_lvl_name}", weight='bold', color='#2c3e50', fontsize=14, pad=15)
        ax2.set_title(f"Precipitation: {active_rain_name}", weight='bold', color='#2c3e50', fontsize=14, pad=15)

        # --- LEFT CHART (WATER LEVEL) ---
        colors_lvl = [get_level_alert(g, unit=u_lvl, t_orange=t_org, t_red=t_red)[0] for g in c_df_lvl['Gradient_1h']]
        ax1.spines['top'].set_visible(False); ax1.spines['right'].set_visible(False)
        ax1.plot(active_df_lvl.index, active_df_lvl['Level'], color='#bdc3c7', alpha=0.4, linestyle='--')
        ax1.plot(c_df_lvl.index, c_df_lvl['Level'], color='#2c3e50', linewidth=3, label=f'Water Level ({u_lvl})')

        ax1.axhline(y=thresh, color='red', linestyle='-.', linewidth=2, label=f'Threshold ({thresh})')

        ax1.set_ylabel(f'Absolute Level ({u_lvl})', weight='bold', fontsize=12)
        lvl_max = active_df_lvl['Level'].max()
        ax1.set_ylim(0, max(lvl_max * 1.2, thresh * 1.1))
        ax1.legend(loc='upper left', fontsize=11)

        ax1_g = ax1.twinx()
        ax1_g.spines['top'].set_visible(False)
        ax1_g.bar(c_df_lvl.index, c_df_lvl['Gradient_1h'], width=0.03, color=colors_lvl, alpha=0.7)
        ax1_g.set_ylabel(f'1h Gradient ({u_lvl}/h)', weight='bold', fontsize=12)

        grad_max = active_df_lvl['Gradient_1h'].max()
        grad_min = active_df_lvl['Gradient_1h'].min()
        ax1_g.set_ylim(min(grad_min * 1.2, -0.5 if u_lvl=='m' else -5), max(grad_max * 1.2, 1.5 if u_lvl=='m' else 35))

        ax1.xaxis.set_major_locator(mdates.HourLocator(interval=6))
        ax1.xaxis.set_major_formatter(mdates.DateFormatter('%d %b %H:%M'))
        ax1.tick_params(axis='x', rotation=45, labelsize=11)
        ax1.tick_params(axis='y', labelsize=11)
        ax1_g.tick_params(axis='y', labelsize=11)

        # --- RIGHT CHART (TRUE INTENSITY: BARS & LINE) ---
        colors_rain = [get_rain_alert(r)[0] for r in c_df_rain['Intensity_1h']]
        ax2.spines['top'].set_visible(False); ax2.spines['right'].set_visible(False)

        ax2.bar(c_df_rain.index, c_df_rain['Intensity_1h'], width=0.03, color=colors_rain, alpha=0.6)
        ax2.plot(active_df_rain.index, active_df_rain['Intensity_1h'], color='#bdc3c7', alpha=0.4, linestyle='--')
        ax2.plot(c_df_rain.index, c_df_rain['Intensity_1h'], color='#2980b9', linewidth=3, label=f'Intensity ({u_rain}/h)')

        ax2.set_ylabel(f'Intensity ({u_rain}/h)', weight='bold', fontsize=12, color='#2c3e50')

        int_max = active_df_rain['Intensity_1h'].max()
        ax2.set_ylim(0, int_max * 1.2 if int_max > 0 else 10)
        ax2.legend(loc='upper left', fontsize=11)

        ax2.xaxis.set_major_locator(mdates.HourLocator(interval=6))
        ax2.xaxis.set_major_formatter(mdates.DateFormatter('%d %b %H:%M'))
        ax2.tick_params(axis='x', rotation=45, labelsize=11)
        ax2.tick_params(axis='y', labelsize=11)

        plt.tight_layout()
        plt.show()

def on_thresh_update(change):
    render_frame()
w_thresh.observe(on_thresh_update, names='value')
slider.observe(render_frame, names='value')

# ==========================================
# 6. ORCHESTRATION
# ==========================================
def on_analyze_clicked(b):
    global active_df_lvl, active_df_rain, active_lvl_name, active_rain_name

    with chart_output: clear_output()
    with kpi_output: clear_output()
    with slider_output: clear_output()

    scenario = SITES[scenario_dropdown.value]
    start_t, end_t = w_start.value, w_end.value

    with map_output:
        clear_output(wait=True)
        print("Initializing Telemetry Search...")

    if scenario['mode'] == 'hardcoded_id':
        lvl_cfg = scenario['level']
        df_lvl = fetch_by_id(lvl_cfg['id'], lvl_cfg['obs_prop'], start_t, end_t, lvl_cfg['kw'], lvl_cfg['conv'], index=lvl_cfg.get('index'))
        active_lvl_name = lvl_cfg['name']
        lat_lvl, lon_lvl = lvl_cfg['lat'], lvl_cfg['lon']

        rain_cfg = scenario['precipitation']
        df_rain = fetch_by_id(rain_cfg['id'], rain_cfg['obs_prop'], start_t, end_t, rain_cfg['kw'], rain_cfg['conv'], index=rain_cfg.get('index'))
        active_rain_name = rain_cfg['name']
        lat_rain, lon_rain = rain_cfg['lat'], rain_cfg['lon']
    else:
        bbox_list = [w_west.value, w_south.value, w_east.value, w_north.value]
        with map_output: print("Scanning BBox for valid Historical Water Level data...")
        df_lvl, active_lvl_name, lat_lvl, lon_lvl = fetch_by_bbox(bbox_list, "water level", start_t, end_t)

        with map_output: print("Scanning BBox for valid Historical Precipitation data...")
        df_rain, active_rain_name, lat_rain, lon_rain = fetch_by_bbox(bbox_list, "precipitation", start_t, end_t)

    if df_lvl.empty or df_rain.empty:
        with chart_output: print("❌ Error: Could not retrieve valid data for both parameters in this area/timeframe.")
        return

    active_df_lvl, active_df_rain = df_lvl, df_rain

    with map_output:
        clear_output(wait=True)
        c_lat, c_lon = (w_south.value + w_north.value) / 2, (w_west.value + w_east.value) / 2
        m = Map(center=(c_lat, c_lon), zoom=10, basemap=basemaps.CartoDB.Positron)
        bounds = ((w_south.value, w_west.value), (w_north.value, w_east.value))
        m.add_layer(Rectangle(bounds=bounds, color='#ff7800', fill_color='#ff7800', fill_opacity=0.1))

        icon_lvl = AwesomeIcon(name='tint', marker_color='white', icon_color='darkblue', spin=False)
        icon_rain = AwesomeIcon(name='cloud', marker_color='white', icon_color='cadetblue', spin=False)
        m.add_layer(Marker(location=(lat_lvl, lon_lvl), title=f"Level: {active_lvl_name}", icon=icon_lvl))
        m.add_layer(Marker(location=(lat_rain, lon_rain), title=f"Rain: {active_rain_name}", icon=icon_rain))
        display(m)

    play.max = len(active_df_lvl) - 1
    slider.max = len(active_df_lvl) - 1
    slider.value = 0

    with slider_output:
        display(widgets.HBox([play, slider], layout=Layout(width='100%', margin='0', padding='10px', background_color='#f8f9fa', border_radius='8px')))

    render_frame()

btn_analyze.on_click(on_analyze_clicked)

# ==========================================
# 7. UI ASSEMBLY
# ==========================================
control_panel = widgets.VBox([
    scenario_dropdown,
    bbox_box,
    w_start, w_end, w_thresh,
    btn_analyze
], layout=Layout(width='32%', padding='15px', background_color='#fdfdfd', border='1px solid #ecf0f1', border_radius='8px'))

map_container = widgets.VBox([map_output], layout=Layout(width='68%', padding='0 0 0 15px'))
top_row = widgets.HBox([control_panel, map_container], layout=Layout(width='100%', margin='0 0 15px 0'))

kpi_container = widgets.VBox([kpi_output], layout=Layout(width='30%', padding='0 15px 0 0'))
charts_container = widgets.VBox([chart_output], layout=Layout(width='70%'))
bottom_row = widgets.HBox([kpi_container, charts_container], layout=Layout(width='100%', margin='10px 0 0 0'))

dashboard = widgets.VBox([header, top_row, time_display, slider_output, bottom_row])
display(dashboard)

# FIXED: Now calls the correct interactive map function
draw_base_map({'new': scenario_dropdown.value})
