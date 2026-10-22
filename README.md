# SPEC — Finanzas personales con bot de Telegram + IA + dashboard

Uso personal, un solo usuario (Matus). Prioridades: coste mínimo de IA, seguridad (nadie más puede usarlo) y cero esfuerzo al registrar gastos.

## 1. Stack

- **Base de datos, auth y backend**: Supabase (Postgres + Auth + Edge Functions en Deno/TypeScript + pg_cron).
- **Bot**: Edge Function `telegram-webhook` que recibe el webhook de Telegram. Librería grammY (versión Deno).
- **IA**: Gemini Flash-Lite (SDK `@google/genai`) con salida estructurada (JSON schema). El proveedor debe ir detrás de una interfaz `LLMProvider` para poder cambiar a GPT-5 nano u otro sin tocar el resto.
- **Dashboard**: Vite + React + TypeScript + Tailwind, desplegado en Vercel. PWA instalable en móvil.
- **Gráficos**: Recharts (o similar ligero).

## 2. Seguridad (obligatorio)

- El bot solo responde al `TELEGRAM_ALLOWED_USER_ID`. Cualquier otro ID se ignora **antes** de llamar a la IA.
- Webhook registrado con `secret_token`; rechazar peticiones sin la cabecera `X-Telegram-Bot-Api-Secret-Token` correcta.
- Supabase Auth con un solo usuario. Registro público desactivado.
- RLS activado en **todas** las tablas. Políticas: solo `auth.uid() = user_id`.
- Claves de IA, token del bot y `service_role` solo como secrets de Edge Functions. En el frontend solo `VITE_SUPABASE_URL` y `VITE_SUPABASE_ANON_KEY`.
- Ningún secreto en el repo. `.env.example` con los nombres de las variables.

## 3. Modelo de datos

- `accounts`: cuentas (nómina, compartida, remunerada/emergencia, broker, cripto...). Campos: nombre, tipo, saldo inicial, moneda (EUR), activa.
- `buckets`: bolsas de presupuesto (gastos fijos, cuenta compartida, ocio/caprichos, emergencia, ahorro, inversión).
- `categories`: lista fija de categorías, cada una asignada a un bucket. Semilla inicial razonable en español (supermercado, restaurantes, transporte, gasolina, suscripciones, hogar, salud, ropa, ocio, regalos, viajes, nómina, otros ingresos...).
- `budgets`: importe mensual por bucket con `valid_from` (histórico de presupuestos; cada mes se compara con el presupuesto vigente en ese mes).
- `transactions`: fecha, importe, tipo (gasto/ingreso/transferencia), cuenta origen/destino, categoría, comercio, nota, `source` (texto/imagen/manual), `raw_input`, `confidence`.
- `merchant_rules`: comercio normalizado → categoría (+ cuenta por defecto opcional). Se crea la primera vez que la IA clasifica un comercio y se actualiza cuando corrijo.
- `instruments`: posiciones de inversión. Campos: nombre, ISIN del fondo real, ticker proxy (ETF que replica el índice), mercado, tipo (fondo/ETF/ETC/cripto).
- `investment_movements`: aportaciones y retiradas (fecha, instrumento, importe en EUR, participaciones calculadas con el precio proxy del día).
- `prices`: precio diario por ticker.
- `monthly_insights`: resultado del análisis mensual de la IA (texto + JSON).

## 4. Flujo del bot

1. Comprobar ID permitido.
2. **Parser sin IA primero** (regex): formatos simples como `12 mercadona`, `+1500 nómina`, `25,50 gasolina cuenta compartida`. Si encaja y el comercio existe en `merchant_rules`, se guarda sin llamar a la IA.
3. Si no encaja → IA con:
   - System prompt corto y fijo que incluya la lista de categorías y cuentas (generada desde la BD, cacheable).
   - Sin historial de conversación.
   - Salida estructurada: array de transacciones (una captura puede traer varias), con `category` restringida a un enum de la lista.
4. **Imágenes**: redimensionar/comprimir antes de enviarlas (ancho máximo ~1024 px). Extraer todas las transacciones de la captura (puede ser de 1 o 2 días de gastos).
5. Detectar duplicados (misma fecha, importe y comercio) y avisar.
6. Responder con resumen de lo guardado + botones inline: ✅ OK · ✏️ Cambiar categoría · 🏦 Cambiar cuenta · 🗑️ Borrar. Las correcciones actualizan `merchant_rules`.
7. Aviso automático al llegar al 80 % y al 100 % de un bucket del mes.

### Comandos

- `/resumen` — gastado vs presupuesto del mes por bucket.
- `/presupuesto <bucket> <importe>` — cambia el presupuesto desde este mes.
- `/invertir <instrumento> <importe>` — registra aportación.
- `/cartera` — valor actual y rentabilidad.
- `/deshacer` — borra el último movimiento.

## 5. Inversiones

- Cada fondo real se sigue con un ETF proxy que replica el mismo índice (no tiene que ser exacto). **Verificar los tickers y la fuente de datos antes de usarlos.** Punto de partida:
  - MSCI World (fondo IE000ZYRH0Q7) → iShares Core MSCI World UCITS ETF (IWDA / EUNL), en EUR.
  - Emergentes (fondo IE000QAZP7L2) → iShares Core MSCI EM IMI (EIMI / IS3N), en EUR.
  - Nasdaq 100 → iShares Nasdaq 100 UCITS ETF (SXRV / CNDX), en EUR.
  - Oro → Amundi Physical Gold ETC (FR0013416716), cotiza en París en EUR: usar su propio precio.
  - Bitcoin → CoinGecko, en EUR.
- Elegir una API de precios gratuita y fiable para ETFs europeos en EUR. Capa `PriceProvider` intercambiable.
- Edge Function `update-prices` programada con pg_cron una vez al día (después del cierre europeo).
- Al registrar una aportación: participaciones = importe / precio proxy del día. Valor actual = participaciones × último precio. Rentabilidad = valor actual − total aportado (en € y %), por instrumento y total.
- Objetivo de inversión mensual como bucket: el dashboard muestra cuánto falta por aportar este mes.

## 6. Análisis mensual con IA

- Edge Function `monthly-insights` vía pg_cron el día 1 de cada mes (y botón manual en el dashboard).
- Enviar **solo datos agregados**: totales por categoría y bucket del mes, media de los 3 meses anteriores, presupuesto vs real, gastos recurrentes detectados por SQL.
- Devuelve JSON: buckets excedidos, categorías que suben, gastos recurrentes, 3–5 propuestas concretas de recorte, y si se cumplió el objetivo de inversión.
- Guardar en `monthly_insights` y enviar resumen por Telegram.

## 7. Dashboard

Páginas:
- **Inicio**: patrimonio total, saldo por cuenta, mes actual (ingresos, gastos, ahorro), barras de progreso por bucket con estado (dentro / cerca / pasado).
- **Movimientos**: tabla filtrable por fecha, cuenta, categoría y bucket; edición y borrado.
- **Presupuesto**: editar importes por bucket (con fecha de inicio), histórico.
- **Inversiones**: valor, aportado, rentabilidad por instrumento y total, gráfico de evolución, reparto actual vs reparto objetivo.
- **Análisis**: últimos insights mensuales.
- **Ajustes**: cuentas, categorías, reglas de comercios.

Diseño: seguir el estilo de las referencias que aporte (Pinterest) y las skills de diseño frontend disponibles. Mobile-first, modo oscuro.

## 8. Coste de tokens — reglas

- Regex y `merchant_rules` antes que IA.
- Cero historial; prompts cortos; salida JSON mínima.
- Imágenes comprimidas.
- Cálculos siempre en SQL/TypeScript, nunca con la IA.
- Registrar tokens usados por llamada en una tabla `llm_usage` para vigilar el gasto.

## 9. Fases

1. Supabase: esquema, RLS, semillas, auth de un usuario.
2. Bot: webhook seguro, parser regex, IA para texto, guardado, botones de corrección.
3. Bot: capturas de pantalla (varias transacciones por imagen) y duplicados.
4. Presupuestos, avisos y comandos.
5. Inversiones: precios diarios, aportaciones, rentabilidad.
6. Dashboard (React) desplegado en Vercel + PWA.
7. Análisis mensual con IA.
8. Revisión de seguridad y de coste.

Cada fase termina con algo que funcione y se pueda probar.
