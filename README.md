
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from datetime import datetime, timedelta
import random

# Configurar semilla para reproducibilidad
np.random.seed(42)
random.seed(42)

# ============================================================
# DATASET: 150 clientes de empresa SaaS ficticia
# ============================================================

n_clients = 150

# Generar IDs de clientes
customer_ids = [f'C{str(i).zfill(3)}' for i in range(1, n_clients + 1)]

# Canales de adquisición
channels = ['Referral', 'Facebook Ads', 'Google Ads', 'LinkedIn', 'Organic', 'WhatsApp', 'Email Campaign']
channel_weights = [0.20, 0.15, 0.15, 0.10, 0.15, 0.15, 0.10]

# Países (para demostrar trilingüismo)
countries = ['Colombia', 'México', 'USA', 'Canadá (QC)', 'Canadá (ON)', 'España', 'Francia', 'Chile', 'Argentina', 'Perú']
country_weights = [0.15, 0.12, 0.18, 0.10, 0.10, 0.08, 0.07, 0.08, 0.07, 0.05]

# Idiomas según país
def get_language(country):
    lang_map = {
        'Colombia': 'Español', 'México': 'Español', 'Chile': 'Español', 
        'Argentina': 'Español', 'Perú': 'Español', 'España': 'Español',
        'USA': 'Inglés', 'Canadá (ON)': 'Inglés',
        'Canadá (QC)': 'Francés', 'Francia': 'Francés'
    }
    return lang_map.get(country, 'Inglés')

# Generar datos
data = {
    'customer_id': customer_ids,
    'country': np.random.choice(countries, n_clients, p=country_weights),
    'language': [],
    'acquisition_channel': np.random.choice(channels, n_clients, p=channel_weights),
    'first_purchase': [],
    'last_purchase': [],
    'revenue_12m': [],
    'cost_of_service': [],
    'support_tickets': [],
    'plan_type': [],
    'payment_status': []
}

# Fechas de compra (últimos 24 meses)
base_date = datetime(2024, 5, 1)
for i in range(n_clients):
    # First purchase: entre 24 y 3 meses atrás
    days_ago_first = random.randint(90, 730)
    first_date = base_date - timedelta(days=days_ago_first)
    
    # Last purchase: entre first_purchase y hoy
    days_since_first = (base_date - first_date).days
    days_ago_last = random.randint(0, min(days_since_first, 90))
    last_date = base_date - timedelta(days=days_ago_last)
    
    data['first_purchase'].append(first_date.strftime('%Y-%m-%d'))
    data['last_purchase'].append(last_date.strftime('%Y-%m-%d'))
    
    # Revenue basado en plan y país (mercados más ricos = más revenue)
    plan = random.choices(['Basic', 'Pro', 'Enterprise'], weights=[0.40, 0.40, 0.20])[0]
    data['plan_type'].append(plan)
    
    base_revenue = {'Basic': 3000, 'Pro': 8000, 'Enterprise': 20000}[plan]
    country_multiplier = {
        'USA': 1.3, 'Canadá (ON)': 1.2, 'Canadá (QC)': 1.1, 'Francia': 1.15,
        'España': 0.9, 'Colombia': 0.7, 'México': 0.75, 'Chile': 0.8,
        'Argentina': 0.65, 'Perú': 0.7
    }[data['country'][i]]
    
    revenue = int(base_revenue * country_multiplier * random.uniform(0.8, 1.3))
    data['revenue_12m'].append(revenue)
    
    # Costo de servicio (40-70% del revenue, más alto para planes complejos)
    cost_ratio = random.uniform(0.35, 0.65) if plan == 'Basic' else random.uniform(0.45, 0.75)
    data['cost_of_service'].append(int(revenue * cost_ratio))
    
    # Tickets de soporte (más tickets = cliente más problemático)
    base_tickets = {'Basic': 5, 'Pro': 15, 'Enterprise': 30}[plan]
    data['support_tickets'].append(int(base_tickets * random.uniform(0.5, 2.0)))
    
    # Idioma
    data['language'].append(get_language(data['country'][i]))
    
    # Estado de pago
    data['payment_status'].append(random.choices(['Al día', 'Retraso 30 días', 'Retraso 60+ días'], weights=[0.85, 0.10, 0.05])[0])

# Crear DataFrame
df = pd.DataFrame(data)

# Calcular métricas derivadas
df['margin'] = df['revenue_12m'] - df['cost_of_service']
df['margin_pct'] = (df['margin'] / df['revenue_12m'] * 100).round(2)
df['ticket_cost'] = df['support_tickets'] * 50  # $50 por ticket
df['total_cost'] = df['cost_of_service'] + df['ticket_cost']
df['net_profit'] = df['revenue_12m'] - df['total_cost']
df['net_profit_pct'] = (df['net_profit'] / df['revenue_12m'] * 100).round(2)

# Customer lifespan en días
df['first_purchase_dt'] = pd.to_datetime(df['first_purchase'])
df['last_purchase_dt'] = pd.to_datetime(df['last_purchase'])
df['customer_lifespan_days'] = (df['last_purchase_dt'] - df['first_purchase_dt']).dt.days

# CAC estimado por canal
cac_map = {'Referral': 0, 'Facebook Ads': 150, 'Google Ads': 200, 'LinkedIn': 300, 
           'Organic': 50, 'WhatsApp': 25, 'Email Campaign': 75}
df['cac'] = df['acquisition_channel'].map(cac_map)

# LTV estimado (simplificado: revenue anual * 2 años estimados)
df['ltv_estimated'] = df['revenue_12m'] * 2
df['ltv_cac_ratio'] = (df['ltv_estimated'] / (df['cac'] + 1)).round(2)  # +1 para evitar división por cero

# SEGMENTACIÓN
def segment_customer(row):
    if row['net_profit'] > 5000 and row['margin_pct'] > 25 and row['payment_status'] == 'Al día':
        return 'Estrella'
    elif row['net_profit'] < -1000 or (row['margin_pct'] < 5 and row['support_tickets'] > 40):
        return 'Problema'
    elif row['margin_pct'] > 15 and row['customer_lifespan_days'] < 180:
        return 'Potencial'
    else:
        return 'Mantenimiento'

df['segment'] = df.apply(segment_customer, axis=1)

# Guardar CSV
df.to_csv('/mnt/agents/output/customer_data.csv', index=False)

print("✅ Dataset creado:")
print(f"   Total clientes: {len(df)}")
print(f"\n   Distribución por segmento:")
print(df['segment'].value_counts())
print(f"\n   Distribución por idioma:")
print(df['language'].value_counts())
print(f"\n   Métricas clave:")
print(f"   - Revenue promedio: ${df['revenue_12m'].mean():,.0f}")
print(f"   - Margen promedio: {df['margin_pct'].mean():.1f}%")
print(f"   - Net profit promedio: ${df['net_profit'].mean():,.0f}")
print(f"   - Clientes con pérdida: {(df['net_profit'] < 0).sum()} ({(df['net_profit'] < 0).mean()*100:.1f}%)")
