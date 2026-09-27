import React, { useEffect, useState, useCallback } from "react";
import {
  SafeAreaView,
  View,
  Text,
  TextInput,
  TouchableOpacity,
  FlatList,
  StyleSheet,
  RefreshControl,
  Modal,
  ScrollView,
  Alert,
  ActivityIndicator,
  Dimensions,
} from "react-native";
import { StatusBar } from "expo-status-bar";
import { LineChart } from "react-native-chart-kit";

import { loadHoldings, saveHoldings, loadAlerts, saveAlerts } from "./src/utils/storage";
import { fetchQuotesBatch, fetchQuote, currencyFor } from "./src/utils/api";
import { analyzePortfolio, simulateDCA } from "./src/utils/ai";

const SCREEN_WIDTH = Dimensions.get("window").width;

export default function App() {
  const [holdings, setHoldings] = useState([]);
  const [alerts, setAlerts] = useState([]); // { symbol, target, direction: 'above'|'below' }
  const [loading, setLoading] = useState(true);
  const [refreshing, setRefreshing] = useState(false);
  const [addVisible, setAddVisible] = useState(false);
  const [detailSymbol, setDetailSymbol] = useState(null);
  const [triggeredAlerts, setTriggeredAlerts] = useState([]);

  useEffect(() => {
    (async () => {
      const [h, a] = await Promise.all([loadHoldings(), loadAlerts()]);
      setHoldings(h);
      setAlerts(a);
      setLoading(false);
      if (h.length) refreshPrices(h);
    })();
  }, []);

  const refreshPrices = useCallback(
    async (list) => {
      const target = list || holdings;
      if (!target.length) return;
      setRefreshing(true);
      const results = await fetchQuotesBatch(target.map((h) => h.symbol));
      const updated = target.map((h) => {
        const r = results.find((x) => x.symbol === h.symbol);
        if (r && r.data) {
          return {
            ...h,
            currentPrice: r.data.price,
            history: r.data.history,
            lastError: null,
          };
        }
        return { ...h, lastError: r?.error || "unknown" };
      });
      setHoldings(updated);
      await saveHoldings(updated);
      checkAlerts(updated);
      setRefreshing(false);
    },
    [holdings, alerts]
  );

  const checkAlerts = (list) => {
    const fired = [];
    alerts.forEach((a) => {
      const h = list.find((x) => x.symbol === a.symbol);
      if (!h || typeof h.currentPrice !== "number") return;
      if (a.direction === "above" && h.currentPrice >= a.target) {
        fired.push(`${a.symbol} ขึ้นถึง ${a.target} แล้ว (ราคาปัจจุบัน ${h.currentPrice})`);
      }
      if (a.direction === "below" && h.currentPrice <= a.target) {
        fired.push(`${a.symbol} ลงมาที่ ${a.target} แล้ว (ราคาปัจจุบัน ${h.currentPrice})`);
      }
    });
    if (fired.length) setTriggeredAlerts(fired);
  };

  const addHolding = async (holding) => {
    const updated = [...holdings, holding];
    setHoldings(updated);
    await saveHoldings(updated);
    setAddVisible(false);
    refreshPrices(updated);
  };

  const removeHolding = async (symbol) => {
    const updated = holdings.filter((h) => h.symbol !== symbol);
    setHoldings(updated);
    await saveHoldings(updated);
  };

  const addAlert = async (alert) => {
    const updated = [...alerts, alert];
    setAlerts(updated);
    await saveAlerts(updated);
  };

  const removeAlert = async (idx) => {
    const updated = alerts.filter((_, i) => i !== idx);
    setAlerts(updated);
    await saveAlerts(updated);
  };

  const totals = holdings.reduce(
    (acc, h) => {
      const cost = h.avgCost * h.shares;
      const value = (h.currentPrice ?? h.avgCost) * h.shares;
      acc.cost += cost;
      acc.value += value;
      return acc;
    },
    { cost: 0, value: 0 }
  );
  const totalPL = totals.value - totals.cost;
  const totalPLPct = totals.cost > 0 ? (totalPL / totals.cost) * 100 : 0;

  if (loading) {
    return (
      <SafeAreaView style={styles.safe}>
        <ActivityIndicator color="#7C9BFF" size="large" style={{ marginTop: 60 }} />
      </SafeAreaView>
    );
  }

  if (detailSymbol) {
    const holding = holdings.find((h) => h.symbol === detailSymbol);
    return (
      <StockDetailScreen
        holding={holding}
        onBack={() => setDetailSymbol(null)}
        onAddAlert={addAlert}
        onRemoveAlert={removeAlert}
        alerts={alerts.filter((a) => a.symbol === detailSymbol)}
      />
    );
  }

  return (
    <SafeAreaView style={styles.safe}>
      <StatusBar style="light" />
      <ScrollView
        refreshControl={
          <RefreshControl refreshing={refreshing} onRefresh={() => refreshPrices()} tintColor="#7C9BFF" />
        }
        contentContainerStyle={{ paddingBottom: 40 }}
      >
        <View style={styles.header}>
          <Text style={styles.title}>AI Portfolio Assistant</Text>
          <Text style={styles.subtitle}>พอร์ตหุ้นไทย 🇹🇭 และหุ้นสหรัฐ 🇺🇸</Text>
        </View>

        <View style={styles.summaryCard}>
          <Text style={styles.summaryLabel}>มูลค่าพอร์ตรวม</Text>
          <Text style={styles.summaryValue}>{fmt(totals.value)}</Text>
          <View style={styles.summaryRow}>
            <SummaryStat label="ต้นทุนรวม" value={fmt(totals.cost)} />
            <SummaryStat
              label="กำไร/ขาดทุน"
              value={`${totalPL >= 0 ? "+" : ""}${fmt(totalPL)}`}
              color={totalPL >= 0 ? "#4ADE80" : "#F87171"}
            />
            <SummaryStat
              label="% รวม"
              value={`${totalPL >= 0 ? "+" : ""}${totalPLPct.toFixed(2)}%`}
              color={totalPL >= 0 ? "#4ADE80" : "#F87171"}
            />
          </View>
        </View>

        {triggeredAlerts.length > 0 && (
          <View style={styles.alertBanner}>
            {triggeredAlerts.map((t, i) => (
              <Text key={i} style={styles.alertBannerText}>🔔 {t}</Text>
            ))}
          </View>
        )}

        <AIInsightsCard holdings={holdings} />

        <View style={styles.sectionHeaderRow}>
          <Text style={styles.sectionHeader}>หุ้นในพอร์ต</Text>
          <TouchableOpacity style={styles.addBtn} onPress={() => setAddVisible(true)}>
            <Text style={styles.addBtnText}>+ เพิ่มหุ้น</Text>
          </TouchableOpacity>
        </View>

        {holdings.length === 0 ? (
          <Text style={styles.empty}>ยังไม่มีหุ้นในพอร์ต แตะ "+ เพิ่มหุ้น" เพื่อเริ่มต้น</Text>
        ) : (
          holdings.map((h) => (
            <StockRow
              key={h.symbol}
              holding={h}
              onPress={() => setDetailSymbol(h.symbol)}
              onDelete={() => confirmDelete(h.symbol, removeHolding)}
            />
          ))
        )}
      </ScrollView>

      <AddStockModal
        visible={addVisible}
        onClose={() => setAddVisible(false)}
        onSubmit={addHolding}
        existingSymbols={holdings.map((h) => h.symbol)}
      />
    </SafeAreaView>
  );
}

function confirmDelete(symbol, onConfirm) {
  Alert.alert("ลบหุ้น", `ต้องการลบ ${symbol} ออกจากพอร์ตหรือไม่?`, [
    { text: "ยกเลิก", style: "cancel" },
    { text: "ลบ", style: "destructive", onPress: () => onConfirm(symbol) },
  ]);
}

function fmt(n) {
  if (typeof n !== "number" || Number.isNaN(n)) return "-";
  return n.toLocaleString(undefined, { maximumFractionDigits: 2 });
}

function SummaryStat({ label, value, color }) {
  return (
    <View style={{ alignItems: "center", flex: 1 }}>
      <Text style={styles.statLabel}>{label}</Text>
      <Text style={[styles.statValue, color && { color }]}>{value}</Text>
    </View>
  );
}

function AIInsightsCard({ holdings }) {
  const notes = analyzePortfolio(holdings);
  return (
    <View style={styles.aiCard}>
      <Text style={styles.aiTitle}>🤖 AI Portfolio Assistant</Text>
      {notes.map((n, i) => (
        <Text key={i} style={styles.aiNote}>• {n}</Text>
      ))}
    </View>
  );
}

function StockRow({ holding, onPress, onDelete }) {
  const cur = holding.currentPrice ?? holding.avgCost;
  const value = cur * holding.shares;
  const cost = holding.avgCost * holding.shares;
  const pl = value - cost;
  const plPct = cost > 0 ? (pl / cost) * 100 : 0;
  const positive = pl >= 0;
  const symbolCurrency = currencyFor(holding.symbol);

  return (
    <TouchableOpacity style={styles.row} onPress={onPress} onLongPress={onDelete}>
      <View style={{ flex: 1 }}>
        <Text style={styles.rowSymbol}>{holding.symbol}</Text>
        <Text style={styles.rowSub}>
          {holding.shares} หุ้น @ {symbolCurrency}{fmt(holding.avgCost)}
        </Text>
        {holding.lastError && <Text style={styles.rowError}>ดึงราคาไม่สำเร็จ</Text>}
      </View>
      <View style={{ alignItems: "flex-end" }}>
        <Text style={styles.rowValue}>{symbolCurrency}{fmt(value)}</Text>
        <Text style={[styles.rowPL, { color: positive ? "#4ADE80" : "#F87171" }]}>
          {positive ? "+" : ""}{fmt(pl)} ({positive ? "+" : ""}{plPct.toFixed(1)}%)
        </Text>
      </View>
    </TouchableOpacity>
  );
}

function AddStockModal({ visible, onClose, onSubmit, existingSymbols }) {
  const [symbol, setSymbol] = useState("");
  const [shares, setShares] = useState("");
  const [avgCost, setAvgCost] = useState("");
  const [checking, setChecking] = useState(false);
  const [error, setError] = useState(null);

  const reset = () => {
    setSymbol("");
    setShares("");
    setAvgCost("");
    setError(null);
  };

  const submit = async () => {
    const sym = symbol.trim().toUpperCase();
    const sharesNum = parseFloat(shares);
    const costNum = parseFloat(avgCost);

    if (!sym) return setError("กรุณาใส่สัญลักษณ์หุ้น เช่น PTT.BK หรือ AAPL");
    if (existingSymbols.includes(sym)) return setError("มีหุ้นนี้ในพอร์ตแล้ว");
    if (!sharesNum || sharesNum <= 0) return setError("กรุณาใส่จำนวนหุ้นที่ถูกต้อง");
    if (!costNum || costNum <= 0) return setError("กรุณาใส่ต้นทุนเฉลี่ยที่ถูกต้อง");

    setChecking(true);
    setError(null);
    try {
      // Validate the symbol resolves before saving it.
      await fetchQuote(sym);
    } catch (e) {
      setChecking(false);
      return setError(`ไม่พบสัญลักษณ์ "${sym}" กรุณาตรวจสอบอีกครั้ง (เช่น PTT.BK, AAPL)`);
    }
    setChecking(false);
    onSubmit({ symbol: sym, shares: sharesNum, avgCost: costNum });
    reset();
  };

  return (
    <Modal visible={visible} animationType="slide" transparent>
      <View style={styles.modalOverlay}>
        <View style={styles.modalCard}>
          <Text style={styles.modalTitle}>เพิ่มหุ้นใหม่</Text>

          <Text style={styles.label}>สัญลักษณ์หุ้น</Text>
          <TextInput
            style={styles.input}
            placeholder="เช่น PTT.BK หรือ AAPL"
            placeholderTextColor="#6B7280"
            autoCapitalize="characters"
            value={symbol}
            onChangeText={setSymbol}
          />

          <Text style={styles.label}>จำนวนหุ้น</Text>
          <TextInput
            style={styles.input}
            placeholder="เช่น 100"
            placeholderTextColor="#6B7280"
            keyboardType="numeric"
            value={shares}
            onChangeText={setShares}
          />

          <Text style={styles.label}>ต้นทุนเฉลี่ยต่อหุ้น</Text>
          <TextInput
            style={styles.input}
            placeholder="เช่น 35.50"
            placeholderTextColor="#6B7280"
            keyboardType="numeric"
            value={avgCost}
            onChangeText={setAvgCost}
          />

          {error && <Text style={styles.errorText}>{error}</Text>}

          <View style={styles.modalBtnRow}>
            <TouchableOpacity
              style={[styles.modalBtn, styles.modalBtnGhost]}
              onPress={() => {
                reset();
                onClose();
              }}
            >
              <Text style={styles.modalBtnGhostText}>ยกเลิก</Text>
            </TouchableOpacity>
            <TouchableOpacity style={styles.modalBtn} onPress={submit} disabled={checking}>
              {checking ? (
                <ActivityIndicator color="#0B0F19" />
              ) : (
                <Text style={styles.modalBtnText}>เพิ่มหุ้น</Text>
              )}
            </TouchableOpacity>
          </View>
        </View>
      </View>
    </Modal>
  );
}

function StockDetailScreen({ holding, onBack, onAddAlert, onRemoveAlert, alerts }) {
  const [monthly, setMonthly] = useState("1000");
  const [months, setMonths] = useState("6");
  const [alertTarget, setAlertTarget] = useState("");
  const [alertDir, setAlertDir] = useState("above");

  if (!holding) {
    return (
      <SafeAreaView style={styles.safe}>
        <TouchableOpacity onPress={onBack} style={{ padding: 16 }}>
          <Text style={styles.backText}>‹ กลับ</Text>
        </TouchableOpacity>
        <Text style={styles.empty}>ไม่พบข้อมูลหุ้นนี้</Text>
      </SafeAreaView>
    );
  }

  const history = holding.history || [];
  const closes = history.map((p) => p.close);
  const cur = holding.currentPrice ?? holding.avgCost;
  const symbolCurrency = currencyFor(holding.symbol);
  const dca =
    closes.length > 0
      ? simulateDCA(parseFloat(monthly) || 0, parseInt(months) || 1, closes)
      : null;

  const chartData =
    closes.length > 1
      ? {
          labels: closes.map((_, i) => (i % 5 === 0 ? `${i}` : "")),
          datasets: [{ data: closes }],
        }
      : null;

  return (
    <SafeAreaView style={styles.safe}>
      <ScrollView contentContainerStyle={{ paddingBottom: 40 }}>
        <TouchableOpacity onPress={onBack} style={{ padding: 16 }}>
          <Text style={styles.backText}>‹ กลับ</Text>
        </TouchableOpacity>

        <View style={styles.header}>
          <Text style={styles.title}>{holding.symbol}</Text>
          <Text style={styles.subtitle}>
            ราคาปัจจุบัน: {symbolCurrency}{fmt(cur)} · ต้นทุนเฉลี่ย: {symbolCurrency}{fmt(holding.avgCost)}
          </Text>
        </View>

        {chartData ? (
          <LineChart
            data={chartData}
            width={SCREEN_WIDTH - 24}
            height={200}
            withDots={false}
            withInnerLines={false}
            chartConfig={{
              backgroundGradientFrom: "#141A2A",
              backgroundGradientTo: "#141A2A",
              decimalPlaces: 2,
              color: () => "#7C9BFF",
              labelColor: () => "#9CA3AF",
            }}
            bezier
            style={{ marginHorizontal: 12, borderRadius: 16 }}
          />
        ) : (
          <Text style={styles.empty}>ยังไม่มีข้อมูลกราฟย้อนหลัง (กดรีเฟรชที่หน้าแรก)</Text>
        )}

        <View style={styles.sectionHeaderRow}>
          <Text style={styles.sectionHeader}>คำนวณ DCA (ประเมินคร่าวๆ)</Text>
        </View>
        <View style={styles.dcaCard}>
          <Text style={styles.label}>ลงทุนต่อเดือน ({symbolCurrency})</Text>
          <TextInput
            style={styles.input}
            keyboardType="numeric"
            value={monthly}
            onChangeText={setMonthly}
          />
          <Text style={styles.label}>จำนวนเดือน</Text>
          <TextInput
            style={styles.input}
            keyboardType="numeric"
            value={months}
            onChangeText={setMonths}
          />
          {dca && (
            <View style={{ marginTop: 8 }}>
              <Text style={styles.dcaResult}>ลงทุนรวม: {symbolCurrency}{fmt(dca.totalInvested)}</Text>
              <Text style={styles.dcaResult}>จำนวนหุ้นที่ได้ (ประมาณ): {fmt(dca.totalShares)}</Text>
              <Text style={styles.dcaResult}>ต้นทุนเฉลี่ยต่อหุ้น (ประมาณ): {symbolCurrency}{fmt(dca.avgCost)}</Text>
              <Text style={styles.dcaNote}>
                * อิงราคาปิดย้อนหลัง 1 เดือนที่มี ใช้เพื่อประกอบการตัดสินใจเท่านั้น ไม่ใช่คำแนะนำการลงทุน
              </Text>
            </View>
          )}
        </View>

        <View style={styles.sectionHeaderRow}>
          <Text style={styles.sectionHeader}>แจ้งเตือนราคา</Text>
        </View>
        <View style={styles.dcaCard}>
          {alerts.map((a, i) => (
            <View key={i} style={styles.alertRow}>
              <Text style={styles.alertRowText}>
                {a.direction === "above" ? "แจ้งเมื่อขึ้นถึง" : "แจ้งเมื่อลงถึง"} {symbolCurrency}{a.target}
              </Text>
              <TouchableOpacity onPress={() => onRemoveAlert(i)}>
                <Text style={styles.deleteLink}>ลบ</Text>
              </TouchableOpacity>
            </View>
          ))}
          <Text style={styles.label}>ราคาเป้าหมาย</Text>
          <TextInput
            style={styles.input}
            keyboardType="numeric"
            value={alertTarget}
            onChangeText={setAlertTarget}
          />
          <View style={styles.dirRow}>
            <TouchableOpacity
              style={[styles.dirBtn, alertDir === "above" && styles.dirBtnActive]}
              onPress={() => setAlertDir("above")}
            >
              <Text style={styles.dirBtnText}>ขึ้นถึง</Text>
            </TouchableOpacity>
            <TouchableOpacity
              style={[styles.dirBtn, alertDir === "below" && styles.dirBtnActive]}
              onPress={() => setAlertDir("below")}
            >
              <Text style={styles.dirBtnText}>ลงถึง</Text>
            </TouchableOpacity>
          </View>
          <TouchableOpacity
            style={styles.modalBtn}
            onPress={() => {
              const t = parseFloat(alertTarget);
              if (!t || t <= 0) return;
              onAddAlert({ symbol: holding.symbol, target: t, direction: alertDir });
              setAlertTarget("");
            }}
          >
            <Text style={styles.modalBtnText}>ตั้งแจ้งเตือน</Text>
          </TouchableOpacity>
          <Text style={styles.dcaNote}>
            * การแจ้งเตือนจะถูกตรวจสอบทุกครั้งที่รีเฟรชราคาในแอป (ไม่ใช่ push notification เบื้องหลัง)
          </Text>
        </View>
      </ScrollView>
    </SafeAreaView>
  );
}

const styles = StyleSheet.create({
  safe: { flex: 1, backgroundColor: "#0B0F19" },
  header: { paddingHorizontal: 20, paddingTop: 8, paddingBottom: 12 },
  title: { color: "#FFFFFF", fontSize: 26, fontWeight: "700" },
  subtitle: { color: "#9CA3AF", fontSize: 13, marginTop: 4 },
  summaryCard: {
    marginHorizontal: 16,
    backgroundColor: "#141A2A",
    borderRadius: 20,
    padding: 20,
  },
  summaryLabel: { color: "#9CA3AF", fontSize: 13 },
  summaryValue: { color: "#FFFFFF", fontSize: 34, fontWeight: "700", marginTop: 4 },
  summaryRow: { flexDirection: "row", marginTop: 16 },
  statLabel: { color: "#6B7280", fontSize: 12 },
  statValue: { color: "#E5E7EB", fontSize: 15, fontWeight: "600", marginTop: 2 },
  alertBanner: {
    marginHorizontal: 16,
    marginTop: 12,
    backgroundColor: "#3B2F1B",
    borderRadius: 14,
    padding: 12,
  },
  alertBannerText: { color: "#FBBF24", fontSize: 13, marginBottom: 2 },
  aiCard: {
    marginHorizontal: 16,
    marginTop: 12,
    backgroundColor: "#141A2A",
    borderRadius: 18,
    padding: 16,
  },
  aiTitle: { color: "#7C9BFF", fontWeight: "700", fontSize: 15, marginBottom: 8 },
  aiNote: { color: "#D1D5DB", fontSize: 13, marginBottom: 6, lineHeight: 18 },
  sectionHeaderRow: {
    flexDirection: "row",
    justifyContent: "space-between",
    alignItems: "center",
    marginHorizontal: 16,
    marginTop: 20,
    marginBottom: 8,
  },
  sectionHeader: { color: "#FFFFFF", fontSize: 16, fontWeight: "700" },
  addBtn: { backgroundColor: "#7C9BFF", borderRadius: 12, paddingHorizontal: 12, paddingVertical: 6 },
  addBtnText: { color: "#0B0F19", fontWeight: "700", fontSize: 13 },
  empty: { color: "#6B7280", textAlign: "center", marginTop: 24, paddingHorizontal: 30 },
  row: {
    flexDirection: "row",
    marginHorizontal: 16,
    marginBottom: 10,
    backgroundColor: "#141A2A",
    borderRadius: 16,
    padding: 14,
    alignItems: "center",
  },
  rowSymbol: { color: "#FFFFFF", fontSize: 16, fontWeight: "700" },
  rowSub: { color: "#9CA3AF", fontSize: 12, marginTop: 2 },
  rowError: { color: "#F87171", fontSize: 11, marginTop: 2 },
  rowValue: { color: "#FFFFFF", fontSize: 15, fontWeight: "600" },
  rowPL: { fontSize: 12, marginTop: 2 },
  modalOverlay: { flex: 1, backgroundColor: "rgba(0,0,0,0.6)", justifyContent: "flex-end" },
  modalCard: { backgroundColor: "#141A2A", borderTopLeftRadius: 24, borderTopRightRadius: 24, padding: 20 },
  modalTitle: { color: "#FFFFFF", fontSize: 18, fontWeight: "700", marginBottom: 12 },
  label: { color: "#9CA3AF", fontSize: 12, marginTop: 10, marginBottom: 4 },
  input: {
    backgroundColor: "#0B0F19",
    borderRadius: 12,
    paddingHorizontal: 14,
    paddingVertical: 10,
    color: "#FFFFFF",
    fontSize: 15,
    borderWidth: 1,
    borderColor: "#232B3E",
  },
  errorText: { color: "#F87171", fontSize: 12, marginTop: 10 },
  modalBtnRow: { flexDirection: "row", marginTop: 20, gap: 10 },
  modalBtn: {
    flex: 1,
    backgroundColor: "#7C9BFF",
    borderRadius: 14,
    paddingVertical: 12,
    alignItems: "center",
    marginTop: 12,
  },
  modalBtnGhost: { backgroundColor: "#232B3E" },
  modalBtnText: { color: "#0B0F19", fontWeight: "700" },
  modalBtnGhostText: { color: "#D1D5DB", fontWeight: "600" },
  backText: { color: "#7C9BFF", fontSize: 16 },
  dcaCard: { marginHorizontal: 16, backgroundColor: "#141A2A", borderRadius: 18, padding: 16 },
  dcaResult: { color: "#D1D5DB", fontSize: 13, marginTop: 4 },
  dcaNote: { color: "#6B7280", fontSize: 11, marginTop: 10, lineHeight: 16 },
  alertRow: {
    flexDirection: "row",
    justifyContent: "space-between",
    alignItems: "center",
    backgroundColor: "#0B0F19",
    borderRadius: 10,
    padding: 10,
    marginBottom: 8,
  },
  alertRowText: { color: "#D1D5DB", fontSize: 13 },
  deleteLink: { color: "#F87171", fontSize: 12 },
  dirRow: { flexDirection: "row", gap: 10, marginTop: 10 },
  dirBtn: {
    flex: 1,
    backgroundColor: "#0B0F19",
    borderRadius: 10,
    paddingVertical: 8,
    alignItems: "center",
    borderWidth: 1,
    borderColor: "#232B3E",
  },
  dirBtnActive: { borderColor: "#7C9BFF", backgroundColor: "#1E2740" },
  dirBtnText: { color: "#D1D5DB", fontSize: 13 },
});
