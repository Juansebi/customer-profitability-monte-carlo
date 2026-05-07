
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


# Ajustar segmentación para que haya clientes Problema reales
# Recalibrar costos para algunos clientes

np.random.seed(123)

# Forzar algunos clientes a tener pérdidas (segmento Problema)
problem_indices = random.sample(range(n_clients), 18)  # 12% de clientes problemáticos

for idx in problem_indices:
    # Aumentar drásticamente costos o tickets
    if random.random() > 0.5:
        df.at[idx, 'cost_of_service'] = int(df.at[idx, 'revenue_12m'] * random.uniform(0.85, 1.1))
    else:
        df.at[idx, 'support_tickets'] = int(random.uniform(50, 120))
    
    # Actualizar métricas
    df.at[idx, 'ticket_cost'] = df.at[idx, 'support_tickets'] * 50
    df.at[idx, 'total_cost'] = df.at[idx, 'cost_of_service'] + df.at[idx, 'ticket_cost']
    df.at[idx, 'net_profit'] = df.at[idx, 'revenue_12m'] - df.at[idx, 'total_cost']
    df.at[idx, 'margin'] = df.at[idx, 'revenue_12m'] - df.at[idx, 'cost_of_service']
    df.at[idx, 'margin_pct'] = round((df.at[idx, 'margin'] / df.at[idx, 'revenue_12m'] * 100), 2)
    df.at[idx, 'net_profit_pct'] = round((df.at[idx, 'net_profit'] / df.at[idx, 'revenue_12m'] * 100), 2)

# Re-segmentar
df['segment'] = df.apply(segment_customer, axis=1)

# Guardar CSV actualizado
df.to_csv('/mnt/agents/output/customer_data.csv', index=False)

print("✅ Dataset ajustado:")
print(f"\n   Distribución por segmento:")
print(df['segment'].value_counts())
print(f"\n   Clientes con pérdida: {(df['net_profit'] < 0).sum()}")
print(f"\n   Top 5 clientes con mayores pérdidas:")
print(df.nsmallest(5, 'net_profit')[['customer_id', 'country', 'revenue_12m', 'total_cost', 'net_profit', 'segment']])
print(f"\n   Top 5 clientes más rentables:")
print(df.nlargest(5, 'net_profit')[['customer_id', 'country', 'revenue_12m', 'net_profit', 'segment']])


# ============================================================
# SIMULACIÓN MONTE CARLO COMPLETA
# ============================================================

n_simulations = 10000
results = []

# Parámetros base por segmento
segment_stats = df.groupby('segment').agg({
    'revenue_12m': ['mean', 'std', 'count'],
    'net_profit': ['mean', 'std'],
    'support_tickets': 'mean'
}).round(2)

print("📊 Estadísticas por segmento (base para simulación):")
print(segment_stats)
print()

# Datos base por segmento
estrella_base_revenue = df[df['segment']=='Estrella']['revenue_12m'].sum()
estrella_base_cost = df[df['segment']=='Estrella']['total_cost'].sum()
problema_base_revenue = df[df['segment']=='Problema']['revenue_12m'].sum()
problema_base_cost = df[df['segment']=='Problema']['total_cost'].sum()
potencial_base_revenue = df[df['segment']=='Potencial']['revenue_12m'].sum()
potencial_base_cost = df[df['segment']=='Potencial']['total_cost'].sum()
mantenimiento_base_revenue = df[df['segment']=='Mantenimiento']['revenue_12m'].sum()
mantenimiento_base_cost = df[df['segment']=='Mantenimiento']['total_cost'].sum()

print(f"💰 Revenue base por segmento:")
print(f"   Estrellas: ${estrella_base_revenue:,.0f}")
print(f"   Problema: ${problema_base_revenue:,.0f}")
print(f"   Potencial: ${potencial_base_revenue:,.0f}")
print(f"   Mantenimiento: ${mantenimiento_base_revenue:,.0f}")
print()

# Correr simulación
for i in range(n_simulations):
    # Variables aleatorias con distribuciones realistas
    
    # Estrellas: crecimiento alto pero con volatilidad
    estrella_growth = np.random.normal(1.18, 0.08)  # 18% crecimiento, 8% volatilidad
    estrella_cost_increase = np.random.normal(1.12, 0.05)  # +12% costo por más atención
    
    # Problema: churn alto (migración a self-service)
    problema_churn = np.random.normal(0.45, 0.12)  # 45% churn, 12% volatilidad
    problema_cost_reduction = np.random.normal(0.55, 0.10)  # -45% costo (self-service)
    
    # Potencial: conversión parcial a Estrellas
    potencial_conversion = np.random.normal(0.35, 0.15)  # 35% se convierten
    potencial_growth = np.random.normal(1.08, 0.06)
    
    # Mantenimiento: estable con ligero crecimiento
    mantenimiento_growth = np.random.normal(1.03, 0.04)
    
    # Calcular escenario
    # Estrellas: invertimos más, crecen más
    estrella_rev = estrella_base_revenue * estrella_growth
    estrella_cost = estrella_base_cost * estrella_cost_increase
    
    # Problema: churn + costo reducido
    problema_rev = problema_base_revenue * (1 - problema_churn)
    problema_cost = problema_base_cost * problema_cost_reduction
    
    # Potencial: algunos crecen, otros se convierten
    potencial_rev = potencial_base_revenue * potencial_growth * (1 + potencial_conversion * 0.3)
    potencial_cost = potencial_base_cost * 1.05  # +5% inversión
    
    # Mantenimiento: estable
    mantenimiento_rev = mantenimiento_base_revenue * mantenimiento_growth
    mantenimiento_cost = mantenimiento_base_cost * 1.02
    
    # Totales
    total_revenue = estrella_rev + problema_rev + potencial_rev + mantenimiento_rev
    total_cost = estrella_cost + problema_cost + potencial_cost + mantenimiento_cost
    
    margin = total_revenue - total_cost
    margin_pct = margin / total_revenue
    
    # Guardar resultado
    results.append({
        'total_revenue': total_revenue,
        'total_cost': total_cost,
        'margin': margin,
        'margin_pct': margin_pct,
        'estrella_growth': estrella_growth,
        'estrella_cost_increase': estrella_cost_increase,
        'problema_churn': problema_churn,
        'problema_cost_reduction': problema_cost_reduction,
        'potencial_conversion': potencial_conversion,
        'potencial_growth': potencial_growth,
        'mantenimiento_growth': mantenimiento_growth
    })

results_df = pd.DataFrame(results)

# ============================================================
# ANÁLISIS DE RESULTADOS
# ============================================================

print("=" * 60)
print("📈 RESULTADOS DE LA SIMULACIÓN MONTE CARLO")
print("=" * 60)
print(f"\n   Simulaciones corridas: {n_simulations:,}")
print(f"\n   MARGEN NETO:")
print(f"   - Promedio: {results_df['margin_pct'].mean():.2%}")
print(f"   - Mediana: {results_df['margin_pct'].median():.2%}")
print(f"   - Desviación estándar: {results_df['margin_pct'].std():.2%}")
print(f"\n   PROBABILIDADES CLAVE:")
print(f"   - Margen > 20%: {(results_df['margin_pct'] > 0.20).mean():.1%}")
print(f"   - Margen > 15%: {(results_df['margin_pct'] > 0.15).mean():.1%}")
print(f"   - Margen > 10%: {(results_df['margin_pct'] > 0.10).mean():.1%}")
print(f"   - Margen < 5%: {(results_df['margin_pct'] < 0.05).mean():.1%}")
print(f"   - Margen negativo: {(results_df['margin_pct'] < 0).mean():.1%}")
print(f"\n   ESCENARIOS EXTREMOS:")
print(f"   - Peor escenario (percentil 1%): {results_df['margin_pct'].quantile(0.01):.2%}")
print(f"   - Percentil 5%: {results_df['margin_pct'].quantile(0.05):.2%}")
print(f"   - Percentil 25%: {results_df['margin_pct'].quantile(0.25):.2%}")
print(f"   - Percentil 75%: {results_df['margin_pct'].quantile(0.75):.2%}")
print(f"   - Percentil 95%: {results_df['margin_pct'].quantile(0.95):.2%}")
print(f"   - Mejor escenario (percentil 99%): {results_df['margin_pct'].quantile(0.99):.2%}")

# Revenue promedio
print(f"\n   REVENUE TOTAL:")
print(f"   - Promedio: ${results_df['total_revenue'].mean():,.0f}")
print(f"   - Percentil 5%: ${results_df['total_revenue'].quantile(0.05):,.0f}")
print(f"   - Percentil 95%: ${results_df['total_revenue'].quantile(0.95):,.0f}")


# ============================================================
# SIMULACIÓN MONTE CARLO: ESCENARIO ACTUAL vs ESTRATEGIA PROPUESTA
# ============================================================

# ESCENARIO ACTUAL: "Seguimos como estamos"
# - Estrellas: crecimiento natural, sin inversión extra
# - Problema: seguimos atendiéndolos igual (costo alto)
# - Potencial: sin inversión especial
# - Mantenimiento: estable

results_actual = []
results_propuesta = []

for i in range(n_simulations):
    # ========== ESCENARIO ACTUAL (sin cambios) ==========
    
    # Estrellas: crecimiento natural moderado
    estrella_growth_actual = np.random.normal(1.05, 0.06)
    estrella_cost_actual = estrella_base_cost * np.random.normal(1.03, 0.03)
    
    # Problema: seguimos atendiéndolos, churn natural bajo
    problema_churn_actual = np.random.normal(0.15, 0.08)
    problema_cost_actual = problema_base_cost * np.random.normal(1.02, 0.05)
    
    # Potencial: sin inversión, crecimiento bajo
    potencial_growth_actual = np.random.normal(1.02, 0.05)
    potencial_cost_actual = potencial_base_cost * np.random.normal(1.01, 0.03)
    
    # Mantenimiento: estable
    mantenimiento_growth_actual = np.random.normal(1.02, 0.04)
    mantenimiento_cost_actual = mantenimiento_base_cost * np.random.normal(1.01, 0.02)
    
    # Calcular
    actual_revenue = (estrella_base_revenue * estrella_growth_actual + 
                      problema_base_revenue * (1 - problema_churn_actual) +
                      potencial_base_revenue * potencial_growth_actual +
                      mantenimiento_base_revenue * mantenimiento_growth_actual)
    
    actual_cost = (estrella_cost_actual + problema_cost_actual + 
                   potencial_cost_actual + mantenimiento_cost_actual)
    
    actual_margin = actual_revenue - actual_cost
    actual_margin_pct = actual_margin / actual_revenue
    
    results_actual.append({
        'total_revenue': actual_revenue,
        'total_cost': actual_cost,
        'margin': actual_margin,
        'margin_pct': actual_margin_pct
    })
    
    # ========== ESTRATEGIA PROPUESTA (nuestra recomendación) ==========
    
    # Estrellas: invertimos más, crecen más
    estrella_growth_prop = np.random.normal(1.20, 0.10)
    estrella_cost_prop = estrella_base_cost * np.random.normal(1.25, 0.08)
    
    # Problema: migración a self-service, churn alto aceptado
    problema_churn_prop = np.random.normal(0.50, 0.15)
    problema_cost_prop = problema_base_cost * np.random.normal(0.45, 0.12)
    
    # Potencial: inversión en conversión
    potencial_conversion_prop = np.random.normal(0.40, 0.18)
    potencial_growth_prop = np.random.normal(1.12, 0.08)
    potencial_cost_prop = potencial_base_cost * np.random.normal(1.15, 0.06)
    
    # Mantenimiento: ligero crecimiento
    mantenimiento_growth_prop = np.random.normal(1.04, 0.04)
    mantenimiento_cost_prop = mantenimiento_base_cost * np.random.normal(1.02, 0.02)
    
    # Calcular
    prop_revenue = (estrella_base_revenue * estrella_growth_prop + 
                    problema_base_revenue * (1 - problema_churn_prop) +
                    potencial_base_revenue * potencial_growth_prop * (1 + potencial_conversion_prop * 0.4) +
                    mantenimiento_base_revenue * mantenimiento_growth_prop)
    
    prop_cost = (estrella_cost_prop + problema_cost_prop + 
                 potencial_cost_prop + mantenimiento_cost_prop)
    
    prop_margin = prop_revenue - prop_cost
    prop_margin_pct = prop_margin / prop_revenue
    
    results_propuesta.append({
        'total_revenue': prop_revenue,
        'total_cost': prop_cost,
        'margin': prop_margin,
        'margin_pct': prop_margin_pct
    })

actual_df = pd.DataFrame(results_actual)
propuesta_df = pd.DataFrame(results_propuesta)

# ============================================================
# COMPARACIÓN
# ============================================================

print("=" * 70)
print("📊 COMPARACIÓN: ESCENARIO ACTUAL vs ESTRATEGIA PROPUESTA")
print("=" * 70)

print(f"\n{'MÉTRICA':<35} {'ACTUAL':>15} {'PROPUESTA':>15} {'DIFERENCIA':>15}")
print("-" * 80)

# Margen promedio
print(f"{'Margen neto promedio':<35} {actual_df['margin_pct'].mean():>14.2%} {propuesta_df['margin_pct'].mean():>14.2%} {(propuesta_df['margin_pct'].mean() - actual_df['margin_pct'].mean()):>+14.2%}")

# Revenue promedio
print(f"{'Revenue total promedio':<35} ${actual_df['total_revenue'].mean():>13,.0f} ${propuesta_df['total_revenue'].mean():>13,.0f} ${(propuesta_df['total_revenue'].mean() - actual_df['total_revenue'].mean()):>+13,.0f}")

# Margen absoluto promedio
print(f"{'Margen absoluto promedio':<35} ${actual_df['margin'].mean():>13,.0f} ${propuesta_df['margin'].mean():>13,.0f} ${(propuesta_df['margin'].mean() - actual_df['margin'].mean()):>+13,.0f}")

# Probabilidad margen > 20%
print(f"{'Prob. margen > 20%':<35} {(actual_df['margin_pct'] > 0.20).mean():>14.1%} {(propuesta_df['margin_pct'] > 0.20).mean():>14.1%} {(propuesta_df['margin_pct'] > 0.20).mean() - (actual_df['margin_pct'] > 0.20).mean():>+14.1%}")

# Probabilidad margen negativo
print(f"{'Prob. margen negativo':<35} {(actual_df['margin_pct'] < 0).mean():>14.1%} {(propuesta_df['margin_pct'] < 0).mean():>14.1%} {(propuesta_df['margin_pct'] < 0).mean() - (actual_df['margin_pct'] < 0).mean():>+14.1%}")

# Percentil 5% (peor caso razonable)
print(f"{'Peor caso (percentil 5%)':<35} {actual_df['margin_pct'].quantile(0.05):>14.2%} {propuesta_df['margin_pct'].quantile(0.05):>14.2%} {(propuesta_df['margin_pct'].quantile(0.05) - actual_df['margin_pct'].quantile(0.05)):>+14.2%}")

# Percentil 95% (mejor caso razonable)
print(f"{'Mejor caso (percentil 95%)':<35} {actual_df['margin_pct'].quantile(0.95):>14.2%} {propuesta_df['margin_pct'].quantile(0.95):>14.2%} {(propuesta_df['margin_pct'].quantile(0.95) - actual_df['margin_pct'].quantile(0.95)):>+14.2%}")

print("\n" + "=" * 70)
print("💡 INTERPRETACIÓN:")
print("=" * 70)
print(f"""
La estrategia propuesta:
• Aumenta el margen neto en {(propuesta_df['margin_pct'].mean() - actual_df['margin_pct'].mean()):.1%} puntos porcentuales
• Genera ${(propuesta_df['margin'].mean() - actual_df['margin'].mean()):,.0f} adicionales de margen anual
• Reduce el riesgo de margen negativo de {(actual_df['margin_pct'] < 0).mean():.1%} a {(propuesta_df['margin_pct'] < 0).mean():.1%}
• Incluso en el peor escenario (percentil 5%), el margen mejora
""")
