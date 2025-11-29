# milk_iot_system
import dash
from dash import dcc, html, Input, Output
import plotly.graph_objs as go
import numpy as np

app = dash.Dash(__name__, title="Цифровой двойник айрана")

def predict_pH(t, dose, variant):
    """
    Базалық аналитикалық модель (A нұсқасы)
    Кесте 7–8–9 логикасына сәйкес.
    """
    if variant == 0:      # Контроль
        a, b = 4.605, 0.125
    elif variant == 1:    # Добавка 1
        a, b = 4.535, 0.102
        a -= 0.02 * dose
    else:                 # Добавка 2
        a, b = 4.506, 0.125
        a -= 0.03 * dose

    return a - b * np.log(t + 1)


# ---------------- UI ----------------
app.layout = html.Div([
    html.H1("ЦИФРОВОЙ ДВОЙНИК АЙРАНА",
            style={'textAlign': 'center', 'color': '#2c3e50'}),

    # Вариант
    html.Div([
        html.Label("Вариант:"),
        dcc.Dropdown(
            id='variant',
            options=[
                {'label': 'Контроль (без добавки)', 'value': 0},
                {'label': 'Добавка 1 (до 3%)', 'value': 1},
                {'label': 'Добавка 2 (до 4%)', 'value': 2}
            ],
            value=0
        )
    ], style={'width': '48%', 'display': 'inline-block'}),

    # Доза
    html.Div([
        html.Label("Доза добавки (%):"),
        dcc.Slider(
            id='dose', min=0, max=4, step=0.1, value=0,
            marks={i: f"{i}%" for i in range(0, 5)}
        )
    ], style={'width': '48%', 'display': 'inline-block', 'padding': '0 20px'}),

    # Время симуляции
    html.Div([
        html.Label("Время симуляции (часы):"),
        dcc.Slider(
            id='time',
            min=0, max=12, step=0.1, value=6,
            marks={i: f'{i}ч' for i in range(0, 13, 2)}
        )
    ], style={'margin': '20px 0'}),

    dcc.Graph(id='ph-graph'),

    html.Div(id='status',
             style={'textAlign': 'center', 'fontSize': 20, 'marginTop': 20})

], style={'fontFamily': 'Arial', 'padding': '20px', 'backgroundColor': '#f9f9f9'})


# ---------------- CALLBACK ----------------
@app.callback(
    [Output('ph-graph', 'figure'),
     Output('status', 'children'),
     Output('status', 'style')],
    [Input('variant', 'value'),
     Input('dose', 'value'),
     Input('time', 'value')]
)
def update_graph(variant, dose, current_time):

    # Таймлайн
    t = np.linspace(0, 12, 500)
    pH = [predict_pH(ti, dose, variant) for ti in t]
    pH_now = predict_pH(current_time, dose, variant)

    # Целевой диапазон (4.3 ± 0.05)
    target_low, target_high = 4.25, 4.35

    # Статусы
    if pH_now <= target_low:
        status = f"СТОП ФЕРМЕНТАЦИЯ! pH = {pH_now:.3f}"
        style = {'color': 'red', 'fontWeight': 'bold'}

    elif target_low < pH_now <= target_high:
        status = f"ЦЕЛЬ ДОСТИГНУТА! pH = {pH_now:.3f} ± 0.05"
        style = {'color': 'green', 'fontWeight': 'bold'}

    else:
        # Оставшееся время (приблизительно)
        remaining = np.exp((4.605 - 4.3) / 0.125) - current_time
        remaining = max(0, remaining)

        status = f"Ферментация... pH = {pH_now:.3f} | Осталось ~{remaining:.1f} ч"
        style = {'color': '#2c3e50'}

    # График
    fig = go.Figure()
    fig.add_trace(go.Scatter(
        x=t, y=pH, mode='lines', name='pH(t)',
        line=dict(width=4, color=['#3498db', '#27ae60', '#e74c3c'][variant])
    ))

    fig.add_trace(go.Scatter(
        x=[current_time], y=[pH_now], mode='markers',
        marker=dict(size=12, color='red'), name='Текущее'
    ))

    # Highlight 4.3 ± 0.05
    fig.add_hrect(
        y0=target_low, y1=target_high,
        fillcolor="green", opacity=0.2,
        annotation_text="Цель: 4.3 ± 0.05", annotation_position="top left"
    )

    fig.update_layout(
        title=f"Динамика pH | {['Контроль','Добавка 1','Добавка 2'][variant]} | Доза: {dose}%",
        xaxis_title="Время (ч)",
        yaxis_title="pH",
        yaxis=dict(range=[4.0, 6.0]),
        template="plotly_white",
        hovermode="x unified"
    )

    return fig, status, style


# ---------------- RUN ----------------
if __name__ == '__main__':
    print("Запуск цифрового двойника айрана...")
    print("Откройте в браузере: http://127.0.0.1:8050")
    app.run(debug=True, port=8050)
