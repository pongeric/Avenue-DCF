# Avenue-DCF
Discount Free Cashflow
<!DOCTYPE html>

<html lang="zh-TW">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>DCF 估值模型工具 - Avenue DCF</title>
  <script src="https://cdn.tailwindcss.com"></script>
</head>
<body>
  <div id="root"></div>
  <script type="module">
    import React, { useState, useEffect } from 'https://esm.sh/react@18.2.0';
    import ReactDOM from 'https://esm.sh/react-dom@18.2.0/client';
    import { Save, Download, Trash2 } from 'https://esm.sh/lucide-react@0.263.1';

```
const DCFValuationTool = () => {
  const [ticker, setTicker] = useState('MRVL');
  
  const [inputs, setInputs] = useState({
    taxRate: 14.0,
    longTermGrowth: 4.0,
    wacc: 12.0,
    sharePrice: 83.45,
    sharesOutstanding: 848,
    historicalYear: 2026,
    historicalRevenue: 8183.3,
    historicalOCF: 2184.8,
    historicalSBC: 611.5,
    historicalCapex: 318.0,
    cash: 2714.5,
    debt: 4777.5
  });

  const [projections, setProjections] = useState([
    { year: 2027, revGrowth: 22.1, ocfMargin: 23.4, sbcGrowth: 24.9, sbcAsRevPct: 7.6, capexGrowth: 57.1, capexAsRevPct: 5.0, fcfMargin: null },
    { year: 2028, revGrowth: 21.3, ocfMargin: 35.0, sbcGrowth: 17.8, sbcAsRevPct: 7.4, capexGrowth: 21.3, capexAsRevPct: 5.0, fcfMargin: null },
    { year: 2029, revGrowth: 18.4, ocfMargin: 25.9, sbcGrowth: 16.7, sbcAsRevPct: 7.3, capexGrowth: 18.4, capexAsRevPct: 5.0, fcfMargin: null },
    { year: 2030, revGrowth: 16.1, ocfMargin: 14.4, sbcGrowth: 19.0, sbcAsRevPct: 7.5, capexGrowth: 16.1, capexAsRevPct: 5.0, fcfMargin: null },
    { year: 2031, revGrowth: 15.0, ocfMargin: 0, sbcGrowth: 0, sbcAsRevPct: 0, capexGrowth: 0, capexAsRevPct: 0, fcfMargin: 20.0 },
    { year: 2032, revGrowth: 14.0, ocfMargin: 0, sbcGrowth: 0, sbcAsRevPct: 0, capexGrowth: 0, capexAsRevPct: 0, fcfMargin: 22.0 }
  ]);

  const [results, setResults] = useState(null);

  useEffect(() => {
    const saved = localStorage.getItem('dcf_models');
    if (saved) {
      try {
        const data = JSON.parse(saved);
        if (data[ticker]) {
          setInputs(data[ticker].inputs);
          setProjections(data[ticker].projections);
        }
      } catch (e) {
        console.error('Error loading saved data');
      }
    }
  }, [ticker]);

  const calculateDCF = () => {
    let revenue = inputs.historicalRevenue;
    let ocf = inputs.historicalOCF;
    let sbc = inputs.historicalSBC;
    let capex = inputs.historicalCapex;
    
    const yearlyData = [];
    let totalPV = 0;

    projections.forEach((proj, idx) => {
      revenue = revenue * (1 + proj.revGrowth / 100);
      
      let fcf;
      
      if (proj.fcfMargin !== null && proj.fcfMargin > 0) {
        fcf = revenue * (proj.fcfMargin / 100);
        ocf = 0;
        sbc = 0;
        capex = 0;
      } else {
        if (proj.ocfMargin > 0) {
          ocf = revenue * (proj.ocfMargin / 100);
        } else {
          ocf = ocf * (1 + proj.revGrowth / 100);
        }
        
        if (proj.sbcGrowth > 0) {
          sbc = sbc * (1 + proj.sbcGrowth / 100);
        } else if (proj.sbcAsRevPct > 0) {
          sbc = revenue * (proj.sbcAsRevPct / 100);
        }
        
        if (proj.capexGrowth > 0) {
          capex = capex * (1 + proj.capexGrowth / 100);
        } else if (proj.capexAsRevPct > 0) {
          capex = revenue * (proj.capexAsRevPct / 100);
        }
        
        fcf = ocf - sbc - capex;
      }
      
      const discountFactor = Math.pow(1 + inputs.wacc / 100, -(idx + 1));
      const pv = fcf * discountFactor;
      
      totalPV += pv;
      
      yearlyData.push({
        year: proj.year,
        revenue: revenue.toFixed(1),
        ocf: proj.fcfMargin ? '-' : ocf.toFixed(1),
        sbc: proj.fcfMargin ? '-' : sbc.toFixed(1),
        capex: proj.fcfMargin ? '-' : capex.toFixed(1),
        fcf: fcf.toFixed(1),
        fcfMargin: proj.fcfMargin ? `${proj.fcfMargin}%` : '-',
        discountFactor: (discountFactor * 100).toFixed(1),
        pv: pv.toFixed(1)
      });
    });

    const lastFCF = parseFloat(yearlyData[yearlyData.length - 1].fcf);
    const terminalValue = (lastFCF * (1 + inputs.longTermGrowth / 100)) / ((inputs.wacc - inputs.longTermGrowth) / 100);
    const pvTerminalValue = terminalValue / Math.pow(1 + inputs.wacc / 100, projections.length);
    
    const enterpriseValue = totalPV + pvTerminalValue;
    const equityValue = enterpriseValue + inputs.cash - inputs.debt;
    const intrinsicValue = equityValue / inputs.sharesOutstanding;
    const marketCap = inputs.sharePrice * inputs.sharesOutstanding;
    const premium = ((intrinsicValue - inputs.sharePrice) / inputs.sharePrice * 100);

    setResults({
      yearlyData,
      totalPV: totalPV.toFixed(1),
      terminalValue: terminalValue.toFixed(1),
      pvTerminalValue: pvTerminalValue.toFixed(1),
      enterpriseValue: enterpriseValue.toFixed(1),
      equityValue: equityValue.toFixed(1),
      intrinsicValue: intrinsicValue.toFixed(2),
      marketCap: marketCap.toFixed(1),
      premium: premium.toFixed(1)
    });
  };

  const saveModel = () => {
    const saved = localStorage.getItem('dcf_models');
    const models = saved ? JSON.parse(saved) : {};
    models[ticker] = { inputs, projections };
    localStorage.setItem('dcf_models', JSON.stringify(models));
    alert('已儲存！');
  };

  const clearModel = () => {
    if (confirm('確定要清除當前模型的數據嗎？')) {
      const saved = localStorage.getItem('dcf_models');
      if (saved) {
        const models = JSON.parse(saved);
        delete models[ticker];
        localStorage.setItem('dcf_models', JSON.stringify(models));
      }
      window.location.reload();
    }
  };

  const exportToCSV = () => {
    if (!results) return;
    
    let csv = 'DCF 估值模型 - ' + ticker + '\n\n';
    csv += '輸入參數\n';
    csv += `股票代號,${ticker}\n`;
    csv += `稅率,${inputs.taxRate}%\n`;
    csv += `永續增長率,${inputs.longTermGrowth}%\n`;
    csv += `WACC,${inputs.wacc}%\n`;
    csv += `當前股價,$${inputs.sharePrice}\n`;
    csv += `流通股數,${inputs.sharesOutstanding}M\n\n`;
    
    csv += '年度預測\n';
    csv += '年份,收入,OCF,股票薪酬,資本支出,自由現金流,FCF Margin,折現率,現值\n';
    results.yearlyData.forEach(row => {
      csv += `${row.year},${row.revenue},${row.ocf},${row.sbc},${row.capex},${row.fcf},${row.fcfMargin},${row.discountFactor}%,${row.pv}\n`;
    });
    
    csv += '\n估值結果\n';
    csv += `自由現金流現值總和,$${results.totalPV}M\n`;
    csv += `終值,$${results.terminalValue}M\n`;
    csv += `終值現值,$${results.pvTerminalValue}M\n`;
    csv += `企業價值,$${results.enterpriseValue}M\n`;
    csv += `股權價值,$${results.equityValue}M\n`;
    csv += `每股內在價值,$${results.intrinsicValue}\n`;
    csv += `市值,$${results.marketCap}M\n`;
    csv += `溢價/折價,${results.premium}%\n`;
    
    const blob = new Blob([csv], { type: 'text/csv' });
    const url = window.URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = `DCF_${ticker}_${new Date().toISOString().split('T')[0]}.csv`;
    a.click();
  };

  return React.createElement('div', { className: "min-h-screen bg-gray-50 p-4" },
    React.createElement('div', { className: "max-w-7xl mx-auto" },
      React.createElement('div', { className: "bg-white rounded-lg shadow-lg p-6 mb-6" },
        React.createElement('div', { className: "flex justify-between items-center mb-6" },
          React.createElement('h1', { className: "text-3xl font-bold text-gray-800" }, 'DCF 估值模型'),
          React.createElement('div', { className: "flex gap-2" },
            React.createElement('button', { onClick: saveModel, className: "flex items-center gap-2 px-4 py-2 bg-green-600 text-white rounded-lg hover:bg-green-700 transition" },
              React.createElement(Save, { className: "w-4 h-4" }), ' 儲存'
            ),
            React.createElement('button', { onClick: exportToCSV, className: "flex items-center gap-2 px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 transition" },
              React.createElement(Download, { className: "w-4 h-4" }), ' 匯出'
            ),
            React.createElement('button', { onClick: clearModel, className: "flex items-center gap-2 px-4 py-2 bg-red-600 text-white rounded-lg hover:bg-red-700 transition" },
              React.createElement(Trash2, { className: "w-4 h-4" }), ' 清除'
            )
          )
        ),

        React.createElement('div', { className: "mb-6" },
          React.createElement('label', { className: "block text-sm font-medium mb-2" }, '股票代號'),
          React.createElement('input', {
            type: "text",
            value: ticker,
            onChange: (e) => setTicker(e.target.value.toUpperCase()),
            className: "px-4 py-2 border rounded-lg w-40 font-semibold text-lg"
          })
        ),

        React.createElement('div', { className: "grid grid-cols-2 md:grid-cols-4 gap-4 mb-6" },
          React.createElement('div', null,
            React.createElement('label', { className: "block text-sm font-medium mb-2" }, '稅率 (%)'),
            React.createElement('input', { type: "number", step: "0.1", value: inputs.taxRate, onChange: (e) => setInputs({...inputs, taxRate: parseFloat(e.target.value)}), className: "w-full px-3 py-2 border rounded-lg" })
          ),
          React.createElement('div', null,
            React.createElement('label', { className: "block text-sm font-medium mb-2" }, '永續增長率 (%)'),
            React.createElement('input', { type: "number", step: "0.1", value: inputs.longTermGrowth, onChange: (e) => setInputs({...inputs, longTermGrowth: parseFloat(e.target.value)}), className: "w-full px-3 py-2 border rounded-lg" })
          ),
          React.createElement('div', null,
            React.createElement('label', { className: "block text-sm font-medium mb-2" }, 'WACC (%)'),
            React.createElement('input', { type: "number", step: "0.1", value: inputs.wacc, onChange: (e) => setInputs({...inputs, wacc: parseFloat(e.target.value)}), className: "w-full px-3 py-2 border rounded-lg" })
          ),
          React.createElement('div', null,
            React.createElement('label', { className: "block text-sm font-medium mb-2" }, '當前股價 ($)'),
            React.createElement('input', { type: "number", step: "0.01", value: inputs.sharePrice, onChange: (e) => setInputs({...inputs, sharePrice: parseFloat(e.target.value)}), className: "w-full px-3 py-2 border rounded-lg" })
          ),
          React.createElement('div', null,
            React.createElement('label', { className: "block text-sm font-medium mb-2" }, '流通股數 (M)'),
            React.createElement('input', { type: "number", value: inputs.sharesOutstanding, onChange: (e) => setInputs({...inputs, sharesOutstanding: parseFloat(e.target.value)}), className: "w-full px-3 py-2 border rounded-lg" })
          ),
          React.createElement('div', null,
            React.createElement('label', { className: "block text-sm font-medium mb-2" }, '現金 ($M)'),
            React.createElement('input', { type: "number", step: "0.1", value: inputs.cash, onChange: (e) => setInputs({...inputs, cash: parseFloat(e.target.value)}), className: "w-full px-3 py-2 border rounded-lg" })
          ),
          React.createElement('div', null,
            React.createElement('label', { className: "block text-sm font-medium mb-2" }, '債務 ($M)'),
            React.createElement('input', { type: "number", step: "0.1", value: inputs.debt, onChange: (e) => setInputs({...inputs, debt: parseFloat(e.target.value)}), className: "w-full px-3 py-2 border rounded-lg" })
          )
        ),

        React.createElement('h2', { className: "text-xl font-bold mb-4" }, '歷史數據 (', inputs.historicalYear, ')'),
        React.createElement('div', { className: "grid grid-cols-2 md:grid-cols-4 gap-4 mb-6" },
          React.createElement('div', null,
            React.createElement('label', { className: "block text-sm font-medium mb-2" }, '收入 ($M)'),
            React.createElement('input', { type: "number", step: "0.1", value: inputs.historicalRevenue, onChange: (e) => setInputs({...inputs, historicalRevenue: parseFloat(e.target.value)}), className: "w-full px-3 py-2 border rounded-lg" })
          ),
          React.createElement('div', null,
            React.createElement('label', { className: "block text-sm font-medium mb-2" }, 'OCF ($M)'),
            React.createElement('input', { type: "number", step: "0.1", value: inputs.historicalOCF, onChange: (e) => setInputs({...inputs, historicalOCF: parseFloat(e.target.value)}), className: "w-full px-3 py-2 border rounded-lg" })
          ),
          React.createElement('div', null,
            React.createElement('label', { className: "block text-sm font-medium mb-2" }, '股票薪酬 ($M)'),
            React.createElement('input', { type: "number", step: "0.1", value: inputs.historicalSBC, onChange: (e) => setInputs({...inputs, historicalSBC: parseFloat(e.target.value)}), className: "w-full px-3 py-2 border rounded-lg" })
          ),
          React.createElement('div', null,
            React.createElement('label', { className: "block text-sm font-medium mb-2" }, '資本支出 ($M)'),
            React.createElement('input', { type: "number", step: "0.1", value: inputs.historicalCapex, onChange: (e) => setInputs({...inputs, historicalCapex: parseFloat(e.target.value)}), className: "w-full px-3 py-2 border rounded-lg" })
          )
        ),

        React.createElement('h2', { className: "text-xl font-bold mb-4" }, '預測數據'),
        React.createElement('div', { className: "overflow-x-auto mb-6" },
          React.createElement('table', { className: "w-full text-sm" },
            React.createElement('thead', { className: "bg-gray-100" },
              React.createElement('tr', null,
                React.createElement('th', { className: "px-3 py-2 text-left" }, '年份'),
                React.createElement('th', { className: "px-3 py-2 text-left" }, '收入增長%'),
                React.createElement('th', { className: "px-3 py-2 text-left" }, 'OCF利潤率%'),
                React.createElement('th', { className: "px-3 py-2 text-left" }, 'SBC增長%'),
                React.createElement('th', { className: "px-3 py-2 text-left" }, 'Capex增長%'),
                React.createElement('th', { className: "px-3 py-2 text-left bg-yellow-100" }, 'FCF Margin%')
              )
            ),
            React.createElement('tbody', null,
              projections.map((proj, idx) =>
                React.createElement('tr', { key: idx, className: "border-b" },
                  React.createElement('td', { className: "px-3 py-2 font-semibold" }, proj.year),
                  React.createElement('td', { className: "px-3 py-2" },
                    React.createElement('input', { type: "number", step: "0.1", value: proj.revGrowth, onChange: (e) => {
                      const newProj = [...projections];
                      newProj[idx].revGrowth = parseFloat(e.target.value);
                      setProjections(newProj);
                    }, className: "w-20 px-2 py-1 border rounded" })
                  ),
                  React.createElement('td', { className: "px-3 py-2" },
                    React.createElement('input', { 
                      type: "number", 
                      step: "0.1", 
                      value: proj.ocfMargin, 
                      onChange: (e) => {
                        const newProj = [...projections];
                        newProj[idx].ocfMargin = parseFloat(e.target.value);
                        setProjections(newProj);
                      }, 
                      className: "w-20 px-2 py-1 border rounded",
                      disabled: proj.fcfMargin !== null && proj.fcfMargin > 0
                    })
                  ),
                  React.createElement('td', { className: "px-3 py-2" },
                    React.createElement('input', { 
                      type: "number", 
                      step: "0.1", 
                      value: proj.sbcGrowth, 
                      onChange: (e) => {
                        const newProj = [...projections];
                        newProj[idx].sbcGrowth = parseFloat(e.target.value);
                        setProjections(newProj);
                      }, 
                      className: "w-20 px-2 py-1 border rounded",
                      disabled: proj.fcfMargin !== null && proj.fcfMargin > 0
                    })
                  ),
                  React.createElement('td', { className: "px-3 py-2" },
                    React.createElement('input', { 
                      type: "number", 
                      step: "0.1", 
                      value: proj.capexGrowth, 
                      onChange: (e) => {
                        const newProj = [...projections];
                        newProj[idx].capexGrowth = parseFloat(e.target.value);
                        setProjections(newProj);
                      }, 
                      className: "w-20 px-2 py-1 border rounded",
                      disabled: proj.fcfMargin !== null && proj.fcfMargin > 0
                    })
                  ),
                  React.createElement('td', { className: "px-3 py-2 bg-yellow-50" },
                    React.createElement('input', { 
                      type: "number", 
                      step: "0.1", 
                      value: proj.fcfMargin || '', 
                      placeholder: "選填",
                      onChange: (e) => {
                        const newProj = [...projections];
                        newProj[idx].fcfMargin = e.target.value ? parseFloat(e.target.value) : null;
                        setProjections(newProj);
                      }, 
                      className: "w-20 px-2 py-1 border rounded bg-yellow-50"
                    })
                  )
                )
              )
            )
          )
        ),
        
        React.createElement('div', { className: "bg-yellow-50 border-l-4 border-yellow-400 p-4 mb-6" },
          React.createElement('p', { className: "text-sm text-gray-700" },
            React.createElement('strong', null, '提示：'),
            '如果填寫了「FCF Margin%」，該年的 OCF、SBC、Capex 將被忽略，直接用收入 × FCF Margin 計算自由現金流。適合用於後期年份的簡化預測。'
          )
        ),

        React.createElement('button', { onClick: calculateDCF, className: "w-full bg-indigo-600 text-white py-3 rounded-lg hover:bg-indigo-700 font-semibold text-lg transition" },
          '計算 DCF'
        )
      ),

      results && React.createElement('div', { className: "bg-white rounded-lg shadow-lg p-6" },
        React.createElement('h2', { className: "text-2xl font-bold mb-4" }, '估值結果'),
        
        React.createElement('div', { className: "overflow-x-auto mb-6" },
          React.createElement('table', { className: "w-full text-sm" },
            React.createElement('thead', { className: "bg-gray-100" },
              React.createElement('tr', null,
                React.createElement('th', { className: "px-3 py-2 text-left" }, '年份'),
                React.createElement('th', { className: "px-3 py-2 text-right" }, '收入'),
                React.createElement('th', { className: "px-3 py-2 text-right" }, 'OCF'),
                React.createElement('th', { className: "px-3 py-2 text-right" }, 'SBC'),
                React.createElement('th', { className: "px-3 py-2 text-right" }, 'Capex'),
                React.createElement('th', { className: "px-3 py-2 text-right bg-yellow-100" }, 'FCF Margin'),
                React.createElement('th', { className: "px-3 py-2 text-right font-semibold" }, 'FCF'),
                React.createElement('th', { className: "px-3 py-2 text-right" }, '折現率'),
                React.createElement('th', { className: "px-3 py-2 text-right font-semibold" }, '現值')
              )
            ),
            React.createElement('tbody', null,
              results.yearlyData.map((row, idx) =>
                React.createElement('tr', { key: idx, className: "border-b hover:bg-gray-50" },
                  React.createElement('td', { className: "px-3 py-2 font-semibold" }, row.year),
                  React.createElement('td', { className: "px-3 py-2 text-right" }, '$', row.revenue),
                  React.createElement('td', { className: "px-3 py-2 text-right" }, row.ocf === '-' ? '-' : `$${row.ocf}`),
                  React.createElement('td', { className: "px-3 py-2 text-right" }, row.sbc === '-' ? '-' : `$(${row.sbc})`),
                  React.createElement('td', { className: "px-3 py-2 text-right" }, row.capex === '-' ? '-' : `$(${row.capex})`),
                  React.createElement('td', { className: "px-3 py-2 text-right bg-yellow-50" }, row.fcfMargin),
                  React.createElement('td', { className: "px-3 py-2 text-right font-bold text-indigo-700" }, '$', row.fcf),
                  React.createElement('td', { className: "px-3 py-2 text-right" }, row.discountFactor, '%'),
                  React.createElement('td', { className: "px-3 py-2 text-right font-bold text-green-700" }, '$', row.pv)
                )
              )
            )
          )
        ),

        React.createElement('div', { className: "grid grid-cols-2 md:grid-cols-3 gap-4 mb-6 text-sm" },
          React.createElement('div', { className: "bg-blue-50 p-4 rounded-lg" },
            React.createElement('div', { className: "text-gray-600 mb-1" }, 'FCF 現值總和'),
            React.createElement('div', { className: "text-xl font-bold" }, '$', results.totalPV, 'M')
          ),
          React.createElement('div', { className: "bg-blue-50 p-4 rounded-lg" },
            React.createElement('div', { className: "text-gray-600 mb-1" }, '終值現值'),
            React.createElement('div', { className: "text-xl font-bold" }, '$', results.pvTerminalValue, 'M')
          ),
          React.createElement('div', { className: "bg-indigo-50 p-4 rounded-lg" },
            React.createElement('div', { className: "text-gray-600 mb-1" }, '企業價值'),
            React.createElement('div', { className: "text-xl font-bold" }, '$', results.enterpriseValue, 'M')
          ),
          React.createElement('div', { className: "bg-indigo-50 p-4 rounded-lg" },
            React.createElement('div', { className: "text-gray-600 mb-1" }, '股權價值'),
            React.createElement('div', { className: "text-xl font-bold" }, '$', results.equityValue, 'M')
          ),
          React.createElement('div', { className: "bg-green-50 p-4 rounded-lg" },
            React.createElement('div', { className: "text-gray-600 mb-1" }, '每股內在價值'),
            React.createElement('div', { className: "text-2xl font-bold text-green-700" }, '$', results.intrinsicValue)
          ),
          React.createElement('div', { className: parseFloat(results.premium) >= 0 ? 'bg-green-50 p-4 rounded-lg' : 'bg-red-50 p-4 rounded-lg' },
            React.createElement('div', { className: "text-gray-600 mb-1" }, '溢價/折價'),
            React.createElement('div', { className: parseFloat(results.premium) >= 0 ? 'text-2xl font-bold text-green-700' : 'text-2xl font-bold text-red-700' },
              results.premium, '%'
            )
          )
        ),

        React.createElement('div', { className: "bg-gray-50 p-4 rounded-lg" },
          React.createElement('div', { className: "text-sm text-gray-600 mb-2" }, '估值總結'),
          React.createElement('div', { className: "text-lg" },
            '當前市價: ',
            React.createElement('span', { className: "font-semibold" }, '$', inputs.sharePrice),
            ' | 內在價值: ',
            React.createElement('span', { className:
```
