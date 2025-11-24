import React, { useState, useEffect, useRef } from 'react';
import { Calculator, Calendar, Plane, FileText, Download, ChevronRight, Edit2, AlertTriangle, AlertCircle, FileX, CalendarCheck, ArrowRight, ArrowLeft, ArrowRightLeft } from 'lucide-react';

// 動態加載 PDF 庫的 Hook
const usePDFLibraries = () => {
  const [loaded, setLoaded] = useState(false);

  useEffect(() => {
    const loadScript = (src) => {
      return new Promise((resolve, reject) => {
        if (document.querySelector(`script[src="${src}"]`)) {
          resolve();
          return;
        }
        const script = document.createElement('script');
        script.src = src;
        script.onload = resolve;
        script.onerror = reject;
        document.head.appendChild(script);
      });
    };

    Promise.all([
      loadScript('https://cdnjs.cloudflare.com/ajax/libs/html2canvas/1.4.1/html2canvas.min.js'),
      loadScript('https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js')
    ]).then(() => {
      setLoaded(true);
    }).catch(err => console.error("Failed to load PDF libs", err));
  }, []);

  return loaded;
};

const App = () => {
  const [activeTab, setActiveTab] = useState('termination');
  const pdfLibsLoaded = usePDFLibraries();

  return (
    <div className="min-h-screen bg-gray-50 text-gray-800 font-sans text-lg">
      <StyleInjector />
      <nav className="bg-blue-600 text-white shadow-lg sticky top-0 z-50">
        <div className="max-w-6xl mx-auto px-4">
          <div className="flex space-x-1 overflow-x-auto md:justify-center py-3 no-scrollbar">
            <NavButton 
              active={activeTab === 'termination'} 
              onClick={() => setActiveTab('termination')}
              icon={<FileX size={20} />} 
              label="終止計算機"
            />
            <NavButton 
              active={activeTab === 'visa'} 
              onClick={() => setActiveTab('visa')}
              icon={<Calendar size={20} />}
              label="4/6/8週計算機"
            />
            <NavButton 
              active={activeTab === 'leave'} 
              onClick={() => setActiveTab('leave')}
              icon={<Plane size={20} />}
              label="放假計算機"
            />
          </div>
        </div>
      </nav>

      <main className="max-w-4xl mx-auto p-4 md:p-6 pb-20">
        {activeTab === 'termination' && <TerminationCalculator pdfLoaded={pdfLibsLoaded} />}
        {activeTab === 'visa' && <VisaCalculator />}
        {activeTab === 'leave' && <LeaveCalculator />}
      </main>
    </div>
  );
};

const NavButton = ({ active, onClick, icon, label }) => (
  <button
    onClick={onClick}
    className={`flex items-center space-x-2 px-4 py-2 rounded-full whitespace-nowrap transition-colors duration-200 text-base ${
      active ? 'bg-white text-blue-600 font-bold shadow-md' : 'text-blue-100 hover:bg-blue-500'
    }`}
  >
    {icon}
    <span>{label}</span>
  </button>
);

// --- Helpers ---
const MS_PER_DAY = 1000 * 60 * 60 * 24;

const parseDate = (dateStr) => {
  if (!dateStr) return null;
  return new Date(dateStr);
};

const formatDate = (date) => {
  if (!date || isNaN(date.getTime())) return '-';
  const d = date.getDate();
  const m = date.getMonth() + 1;
  const y = date.getFullYear();
  return `${d}-${m}-${y}`;
};

// Formatter: Truncate to 1 decimal place (No Rounding), add commas
const formatMoney1DecStrict = (val) => {
  if (val === undefined || val === null || val === '' || isNaN(val)) return '0.0';
  const num = parseFloat(val);
  const truncated = Math.floor(num * 10) / 10;
  return truncated.toLocaleString('en-HK', { minimumFractionDigits: 1, maximumFractionDigits: 1 });
};

// Formatter: Truncate to 2 decimal places (No Rounding)
const formatMoney2Dec = (val) => {
  if (val === undefined || val === null || val === '' || isNaN(val)) return 'HK$0.00';
  const num = parseFloat(val);
  const truncated = Math.floor(num * 100) / 100;
  return 'HK$' + truncated.toLocaleString('en-HK', { minimumFractionDigits: 2, maximumFractionDigits: 2 });
};

// Helper to remove commas for calculation
const parseFormattedNumber = (str) => {
    if (typeof str === 'number') return str;
    if (!str) return 0;
    return parseFloat(str.replace(/,/g, '')) || 0;
};

// --- 1. 終止計數機 (Termination Calculator) ---
const TerminationCalculator = ({ pdfLoaded }) => {
  const [step, setStep] = useState(1);
  const [isNoContractPDF, setIsNoContractPDF] = useState(false);
  
  // Error state for validation
  const [formErrors, setFormErrors] = useState({});
  const [generalError, setGeneralError] = useState('');

  const [inputs, setInputs] = useState({
    // Part 1
    salary: '',
    lastPayPeriodEnd: '',
    lastWorkingDay: '',
    firstDayLastContract: '',
    contractCount: '',
    takenLeave: '',
    untakenStatutoryHolidays: '',
    hasNoticePay: false,
    noticeDate: '',
    hasLongService: false,
    firstDayAllContracts: '',
    transportAllowance: 100,
    ticketType: '', // 'will_buy', 'bought', 'cash'
    ticketCashAmount: '',
    otherPayment: '', 
    calcRemark: '', // New remark field in calculation section
    
    // Part 2
    employerName: '',
    fdhName: '',
    fdhId: '',
    contractNo: '',
    paymentDate: '',
    paymentMethod: 'cash',
    remark: ''
  });

  const [editResults, setEditResults] = useState(null);
  const [metaResults, setMetaResults] = useState(null);

  const resultRef = useRef(null);
  const contractSectionRef = useRef(null);

  // Clear errors when input changes
  const handleInputChange = (field, value) => {
      setInputs(prev => ({ ...prev, [field]: value }));
      if (formErrors[field]) {
          setFormErrors(prev => {
              const newErrors = { ...prev };
              delete newErrors[field];
              return newErrors;
          });
      }
      setGeneralError(''); 
  };

  const handleTransportKeyDown = (e) => {
    if (e.key === 'ArrowUp') {
      e.preventDefault();
      handleInputChange('transportAllowance', (parseFloat(inputs.transportAllowance) || 0) + 100);
    } else if (e.key === 'ArrowDown') {
      e.preventDefault();
      const val = (parseFloat(inputs.transportAllowance) || 0) - 100;
      handleInputChange('transportAllowance', val < 0 ? 0 : val);
    }
  };

  const getDailyRate = () => {
    const salary = parseFloat(inputs.salary) || 0;
    return (salary * 12) / 365;
  };

  const getStartOfTermMonth = () => {
      if(!inputs.lastPayPeriodEnd) return null;
      const d = parseDate(inputs.lastPayPeriodEnd);
      d.setDate(d.getDate() + 1);
      return d;
  };

  const getDaysInTermMonth = () => {
      const start = getStartOfTermMonth();
      const end = parseDate(inputs.lastWorkingDay);
      if (!start || !end) return null;
      return Math.ceil((end - start) / MS_PER_DAY) + 1;
  };

  // Check for Payment Delay Warning (Month + 7 Days logic)
  const getPaymentDelayWarning = () => {
      if (!inputs.lastWorkingDay || !inputs.lastPayPeriodEnd) return null;
      const lastWork = parseDate(inputs.lastWorkingDay);
      const lastPayEnd = parseDate(inputs.lastPayPeriodEnd);
      
      // Calculate: LastPayPeriodEnd + Days in that month + 7
      // "完整月份的日數加7天" implies checking the gap relative to a month structure
      // Prompt Example: "2月28號+8日" -> March 8th.
      // Let's calculate the date that is 7 days after the "Full Month" date.
      
      // First, find +1 Month from LastPayPeriodEnd
      const nextMonthDate = new Date(lastPayEnd);
      nextMonthDate.setMonth(nextMonthDate.getMonth() + 1);
      
      // Add 7 days to that
      const thresholdDate = new Date(nextMonthDate);
      thresholdDate.setDate(thresholdDate.getDate() + 7);

      if (lastWork > thresholdDate) {
          return "提醒：出糧不能遲多過7天，完整月份的日數加7天，例如：2月28號+8日就已經出糧遲多過7天。所以應先出糧其中的月薪，餘下的日數才填寫在「上個支薪期的工資結算日」";
      }
      return null;
  };

  const getLSPDuration = () => {
     if(!inputs.hasLongService || !inputs.firstDayAllContracts || !inputs.lastWorkingDay) return null;
     const start = parseDate(inputs.firstDayAllContracts);
     const end = parseDate(inputs.lastWorkingDay);
     const diffTime = end - start;
     const diffYears = diffTime / (MS_PER_DAY * 365);
     const diffDays = Math.ceil(diffTime / MS_PER_DAY) + 1; 
     return { diffYears, diffDays };
  };

  const getProRataInfo = () => {
    if(!inputs.lastWorkingDay || !inputs.firstDayLastContract) return null;
    const start = parseDate(inputs.firstDayLastContract);
    const end = parseDate(inputs.lastWorkingDay);
    const diffDays = Math.ceil((end - start) / MS_PER_DAY) + 1;

    let entitlement = 14;
    const count = parseInt(inputs.contractCount);
    if (count === 1) entitlement = 14;
    else if (count === 2) entitlement = 17;
    else if (count === 3) entitlement = 21;
    else if (count === 4) entitlement = 25;
    else if (count >= 5) entitlement = 28;

    const proRataDays = (diffDays / 730) * entitlement;
    return { diffDays, entitlement, proRataDays };
  };

  const getFinalLeaveDays = () => {
      const info = getProRataInfo();
      if (!info) return null;
      const taken = parseFloat(inputs.takenLeave) || 0;
      const untakenStatutory = parseFloat(inputs.untakenStatutoryHolidays) || 0;
      return info.proRataDays - taken + untakenStatutory;
  };

  const handleCalculate = () => {
    const errors = {};
    const requiredCalcFields = [
        'salary', 'lastPayPeriodEnd', 'lastWorkingDay', 'firstDayLastContract', 
        'contractCount', 'takenLeave', 'untakenStatutoryHolidays', 'ticketType'
    ];
    
    for (let field of requiredCalcFields) {
      if (inputs[field] === '' || inputs[field] === undefined) {
        errors[field] = true;
      }
    }
    
    if (inputs.transportAllowance === '') errors['transportAllowance'] = true;
    if (inputs.ticketType === 'cash' && inputs.ticketCashAmount === '') errors['ticketCashAmount'] = true;
    if (inputs.hasNoticePay && !inputs.noticeDate) errors['noticeDate'] = true;
    if (inputs.hasLongService && !inputs.firstDayAllContracts) errors['firstDayAllContracts'] = true;

    const salaryNum = parseFloat(inputs.salary) || 0;
    if (inputs.salary !== '' && salaryNum < 4870) {
      errors['salary'] = 'min';
    }

    if (Object.keys(errors).length > 0) {
        setFormErrors(errors);
        if (errors['salary'] === 'min') {
             setGeneralError("合約月薪要多過4870，請檢查和填寫正確月薪");
        } else {
             setGeneralError("你漏寫資料，請檢查和填寫所有資料");
        }
        window.scrollTo({ top: 0, behavior: 'smooth' });
        return;
    }

    setFormErrors({});
    setGeneralError('');

    // --- Calculation Logic ---
    const dLastPayEnd = parseDate(inputs.lastPayPeriodEnd);
    const dLastWork = parseDate(inputs.lastWorkingDay);
    const dFirstDayTermMonth = new Date(dLastPayEnd);
    dFirstDayTermMonth.setDate(dLastPayEnd.getDate() + 1);

    const dailyRate = getDailyRate();

    // 2. Final Salary
    let finalSalary = 0;
    const dCheckLastWork = new Date(dLastWork); dCheckLastWork.setDate(dLastWork.getDate() + 1);
    const dCheckLastPay = new Date(dLastPayEnd); dCheckLastPay.setDate(dLastPayEnd.getDate() + 1);
    const isFullMonth = dCheckLastWork.getDate() === dCheckLastPay.getDate();

    if (isFullMonth) {
        finalSalary = salaryNum;
    } else {
        const totalLastWorkDays = Math.ceil((dLastWork - dLastPayEnd) / MS_PER_DAY);
        const daysInBasisMonth = new Date(dFirstDayTermMonth.getFullYear(), dFirstDayTermMonth.getMonth() + 1, 0).getDate();

        if (totalLastWorkDays > daysInBasisMonth) {
             const remainderDays = totalLastWorkDays - daysInBasisMonth;
             finalSalary = (remainderDays * dailyRate) + salaryNum;
        } else {
             finalSalary = totalLastWorkDays * dailyRate;
        }
    }

    // 3. Untaken Leave Pay
    // Formula: (ProRataDays - Taken + UntakenStatutory) * DailyRate
    const finalLeaveDays = getFinalLeaveDays() || 0;
    const untakenLeavePay = Math.max(0, finalLeaveDays * dailyRate);

    // 4. Notice Pay
    let noticePay = 0;
    if (inputs.hasNoticePay) {
        const dNotice = parseDate(inputs.noticeDate);
        if (dNotice.getTime() === dLastWork.getTime()) {
            noticePay = salaryNum;
        } else {
            const diffDays = Math.ceil((dLastWork - dNotice) / MS_PER_DAY);
            const dCheckNotice = new Date(dNotice);
            dCheckNotice.setMonth(dCheckNotice.getMonth() + 1);
            
            if (dCheckNotice.getDate() === dLastWork.getDate() && dCheckNotice.getMonth() === dLastWork.getMonth()) {
                 noticePay = salaryNum;
            } else {
                 noticePay = diffDays * dailyRate;
            }
        }
    }

    // 5. Long Service Pay
    let longServicePay = 0;
    if (inputs.hasLongService && inputs.firstDayAllContracts) {
        const lspInfo = getLSPDuration();
        if (lspInfo) {
            if (lspInfo.diffYears < 5) {
                longServicePay = 0;
            } else {
                longServicePay = salaryNum * (2/3) * (lspInfo.diffDays / 365);
            }
        }
    }

    // 6. Others
    const transport = parseFloat(inputs.transportAllowance) || 0;
    const other = parseFloat(inputs.otherPayment) || 0;
    let ticketCost = 0;
    let ticketText = "";
    
    if (inputs.ticketType === 'cash') {
        ticketCost = parseFloat(inputs.ticketCashAmount) || 0;
    } else if (inputs.ticketType === 'bought') {
        ticketText = "air ticket already bought";
    } else if (inputs.ticketType === 'will_buy') {
        ticketText = "preparing to buy air ticket";
    }

    // Populate Edit Results
    setEditResults({
        finalSalary: formatMoney1DecStrict(finalSalary),
        untakenLeavePay: formatMoney1DecStrict(untakenLeavePay),
        transport: formatMoney1DecStrict(transport),
        ticketCost: ticketText ? ticketText : formatMoney1DecStrict(ticketCost),
        noticePay: formatMoney1DecStrict(noticePay),
        longServicePay: formatMoney1DecStrict(longServicePay),
        other: formatMoney1DecStrict(other),
    });

    setMetaResults({
        dFirstDayTermMonth,
        dLastWork,
        finalLeaveDays: Math.floor(finalLeaveDays * 10) / 10
    });

    setStep(2);
    setTimeout(() => {
        document.getElementById('result-section')?.scrollIntoView({ behavior: 'smooth' });
    }, 100);
  };

  const handleManualChange = (field, value) => {
      setEditResults(prev => ({
          ...prev,
          [field]: value
      }));
  };

  const calculateTotal = () => {
      if (!editResults) return "0.0";
      // Sum numeric values
      const sum = [
          editResults.finalSalary,
          editResults.untakenLeavePay,
          editResults.transport,
          editResults.noticePay,
          editResults.longServicePay,
          editResults.other,
          // Handle Ticket: only add if it's numeric (not text)
          (!isNaN(parseFormattedNumber(editResults.ticketCost)) && inputs.ticketType === 'cash') ? editResults.ticketCost : 0
      ].reduce((acc, val) => acc + parseFormattedNumber(val), 0);
      
      return formatMoney1DecStrict(sum);
  };

  const handleShowContractForm = () => {
    setStep(3);
    setTimeout(() => {
        contractSectionRef.current?.scrollIntoView({ behavior: 'smooth' });
    }, 100);
  };

  const generatePDF = async (noContract) => {
    setIsNoContractPDF(noContract);
    setFormErrors({});

    if (!noContract) {
        const requiredContractFields = ['employerName', 'fdhName', 'fdhId', 'contractNo', 'paymentDate'];
        const errors = {};
        for (let field of requiredContractFields) {
            if (!inputs[field]) {
                errors[field] = true;
            }
        }
        if (Object.keys(errors).length > 0) {
            setFormErrors(errors);
            setGeneralError("你漏寫資料，請檢查和填寫所有資料");
            contractSectionRef.current?.scrollIntoView({ behavior: 'smooth' });
            return;
        }
    }

    if (!editResults || !resultRef.current || !window.jspdf || !window.html2canvas) return;
    
    const element = resultRef.current;
    element.classList.add('pdf-mode');
    if (noContract) {
        element.classList.add('no-contract-mode');
    }

    try {
      await new Promise(r => setTimeout(r, 100));

      const canvas = await window.html2canvas(element, { scale: 2, useCORS: true });
      const imgData = canvas.toDataURL('image/png');
      
      const { jsPDF } = window.jspdf;
      const doc = new jsPDF('p', 'mm', 'a4');
      const pdfWidth = doc.internal.pageSize.getWidth();
      const pdfHeight = (canvas.height * pdfWidth) / canvas.width;
      
      doc.addImage(imgData, 'PNG', 0, 0, pdfWidth, pdfHeight);
      
      let filename = 'Final_Payment.pdf';
      if (inputs.contractNo) {
          filename = `${inputs.contractNo}_Final_Payment.pdf`;
      }
      doc.save(filename);
    } catch (error) {
      console.error("PDF Gen Error", error);
      alert("PDF 生成失敗，請重試");
    } finally {
        element.classList.remove('pdf-mode');
        element.classList.remove('no-contract-mode');
        setIsNoContractPDF(false);
    }
  };

  return (
    <div className="space-y-8">
      {/* Part 1: Calculation Data */}
      <div className="bg-white p-8 rounded-xl shadow-sm border border-gray-200">
        <h2 className="text-2xl font-bold text-gray-800 border-b pb-4 mb-6 flex items-center gap-2">
          <FileText className="text-blue-600"/> 填寫計算資料
        </h2>
        
        {/* Validation Banner */}
        {generalError && (
             <div className="mb-6 bg-red-100 border-l-4 border-red-500 text-red-700 p-4 rounded flex items-center gap-2 animate-fade-in border border-red-200">
                 <AlertCircle size={24} />
                 <p className="font-bold">{generalError}</p>
             </div>
        )}

        <div className="grid grid-cols-1 gap-6">
          <InputGroup label="合約月薪 (不用填寫$)" error={formErrors.salary}>
            <input 
                type="number" 
                className={`input-field ${formErrors.salary ? 'border-red-500 bg-red-50' : ''}`} 
                value={inputs.salary} 
                onChange={e => handleInputChange('salary', e.target.value)} 
                placeholder="最少：4870" 
            />
            <div className="mt-2 bg-gray-100 border border-gray-200 p-2 rounded text-gray-800 font-medium bg-opacity-50 border-opacity-50" style={{backgroundColor: '#e0e7ff', borderColor: '#c7d2fe'}}>
                 日薪： <span className="font-bold">{formatMoney2Dec(getDailyRate())}</span> (合約月薪 × 12 ÷ 365)
            </div>
          </InputGroup>

          <InputGroup label="上個支薪期的工資結算日" error={formErrors.lastPayPeriodEnd}>
            <input 
                type="date" 
                className={`input-field ${formErrors.lastPayPeriodEnd ? 'border-red-500 bg-red-50' : ''}`} 
                value={inputs.lastPayPeriodEnd} 
                onChange={e => handleInputChange('lastPayPeriodEnd', e.target.value)} 
            />
            <div className="mt-2 p-2 rounded text-gray-800 font-medium" style={{backgroundColor: '#e0e7ff', borderColor: '#c7d2fe', borderWidth: '1px'}}>
                離職月份的支薪期的第一天： <span className="font-bold">{getStartOfTermMonth() ? formatDate(getStartOfTermMonth()) : '-'}</span>
            </div>
          </InputGroup>

          <InputGroup label="最後工作日 (離職日)" error={formErrors.lastWorkingDay}>
            <input 
                type="date" 
                className={`input-field ${formErrors.lastWorkingDay ? 'border-red-500 bg-red-50' : ''}`} 
                value={inputs.lastWorkingDay} 
                onChange={e => handleInputChange('lastWorkingDay', e.target.value)} 
            />
            <div className="mt-2 p-2 rounded text-gray-800 font-medium" style={{backgroundColor: '#e0e7ff', borderColor: '#c7d2fe', borderWidth: '1px'}}>
                離職月份的工作日數： <span className="font-bold">{getDaysInTermMonth() !== null ? getDaysInTermMonth() + ' 日' : '-'}</span>
            </div>
            {getPaymentDelayWarning() && (
                <div className="mt-2 flex items-start gap-2 text-orange-700 bg-orange-50 p-4 rounded border border-orange-200">
                    <AlertTriangle size={28} className="shrink-0 mt-0.5"/>
                    <span className="text-xl font-bold leading-tight">{getPaymentDelayWarning()}</span>
                </div>
            )}
          </InputGroup>

          <InputGroup label="最後一份合約的第一天工作日" error={formErrors.firstDayLastContract}>
            <input 
                type="date" 
                className={`input-field ${formErrors.firstDayLastContract ? 'border-red-500 bg-red-50' : ''}`} 
                value={inputs.firstDayLastContract} 
                onChange={e => handleInputChange('firstDayLastContract', e.target.value)} 
            />
          </InputGroup>

          <InputGroup label="這是第幾份合約 (前後總共有幾多份合約)" error={formErrors.contractCount}>
             <select 
                className={`input-field ${formErrors.contractCount ? 'border-red-500 bg-red-50' : ''}`} 
                value={inputs.contractCount} 
                onChange={e => handleInputChange('contractCount', e.target.value)}
             >
               <option value="" disabled>請選擇</option>
               {[...Array(9).keys()].map(i => <option key={i+1} value={i+1}>{i+1} 份</option>)}
               {[...Array(10).keys()].map(i => <option key={i+11} value={i+11}>{i+11} 份</option>)}
             </select>
             {getProRataInfo() && (
                 <div className="mt-2 p-2 rounded text-gray-800 font-medium" style={{backgroundColor: '#e0e7ff', borderColor: '#c7d2fe', borderWidth: '1px'}}>
                     享有<span className="text-red-600 font-bold">年假</span>日數 (按比例)： <span className="font-bold">{getProRataInfo().proRataDays.toFixed(2)} 日</span>
                     {(inputs.takenLeave || inputs.untakenStatutoryHolidays) && (
                         <div className="mt-1 border-t border-blue-300 pt-1">
                             享有<span className="text-red-600 font-bold">年假+假期</span>日數 (最終)： <span className="font-bold">{getFinalLeaveDays().toFixed(2)} 日</span>
                         </div>
                     )}
                 </div>
             )}
          </InputGroup>

          <InputGroup label={<span>已放<span className="text-red-600">年假</span>日數</span>} error={formErrors.takenLeave}>
            <input 
                type="number" 
                className={`input-field ${formErrors.takenLeave ? 'border-red-500 bg-red-50' : ''}`} 
                value={inputs.takenLeave} 
                onChange={e => handleInputChange('takenLeave', e.target.value)} 
            />
          </InputGroup>

          <InputGroup label={<span>未放<span className="text-red-600">假期</span>日數 (即是休息日和勞工假期)</span>} error={formErrors.untakenStatutoryHolidays}>
            <input 
                type="number" 
                className={`input-field ${formErrors.untakenStatutoryHolidays ? 'border-red-500 bg-red-50' : ''}`} 
                value={inputs.untakenStatutoryHolidays} 
                onChange={e => handleInputChange('untakenStatutoryHolidays', e.target.value)} 
            />
          </InputGroup>

          <InputGroup label="交通津貼 (最少：100) 不用填寫$" error={formErrors.transportAllowance}>
            <input 
              type="number" 
              className={`input-field ${formErrors.transportAllowance ? 'border-red-500 bg-red-50' : ''}`} 
              value={inputs.transportAllowance} 
              onChange={e => {
                 const val = parseFloat(e.target.value);
                 handleInputChange('transportAllowance', isNaN(val) ? '' : Math.max(0, val));
              }}
              onKeyDown={handleTransportKeyDown}
              placeholder="最少：100 (不用填寫$)"
            />
          </InputGroup>
          
          <InputGroup label="回國機票" error={formErrors.ticketType || formErrors.ticketCashAmount}>
            <select 
                className={`input-field mb-2 ${formErrors.ticketType ? 'border-red-500 bg-red-50' : ''}`} 
                value={inputs.ticketType} 
                onChange={e => handleInputChange('ticketType', e.target.value)}
            >
              <option value="" disabled>請選擇</option>
              <option value="will_buy">會買機票</option>
              <option value="bought">已買機票</option>
              <option value="cash">折現機票</option>
            </select>
            {inputs.ticketType === 'cash' && (
              <input 
                type="number" 
                placeholder="金額" 
                className={`input-field ${formErrors.ticketCashAmount ? 'border-red-500 bg-red-50' : ''}`} 
                value={inputs.ticketCashAmount} 
                onChange={e => handleInputChange('ticketCashAmount', e.target.value)} 
              />
            )}
          </InputGroup>

          <div className="border p-4 rounded-lg bg-gray-50">
            <div className="flex items-center justify-between mb-2">
              <label className="font-semibold text-gray-700">一個月代通知金 (如有)</label>
              <input type="checkbox" className="w-6 h-6" checked={inputs.hasNoticePay} onChange={e => handleInputChange('hasNoticePay', e.target.checked)} />
            </div>
            {inputs.hasNoticePay && (
              <div className="grid grid-cols-1 gap-4 mt-2">
                 <InputGroup label="終止合約通知日期" error={formErrors.noticeDate}>
                    <input 
                        type="date" 
                        className={`input-field ${formErrors.noticeDate ? 'border-red-500 bg-red-50' : ''}`} 
                        value={inputs.noticeDate} 
                        onChange={e => handleInputChange('noticeDate', e.target.value)} 
                    />
                 </InputGroup>
              </div>
            )}
          </div>

          <div className="border p-4 rounded-lg bg-gray-50">
            <div className="flex items-center justify-between mb-2">
              <label className="font-semibold text-gray-700">長期服務金 (如有)</label>
              <input type="checkbox" className="w-6 h-6" checked={inputs.hasLongService} onChange={e => handleInputChange('hasLongService', e.target.checked)} />
            </div>
            {inputs.hasLongService && (
               <>
                <InputGroup label="第一份合約的第一天工作日 (所有合約)" error={formErrors.firstDayAllContracts}>
                    <input 
                        type="date" 
                        className={`input-field ${formErrors.firstDayAllContracts ? 'border-red-500 bg-red-50' : ''}`} 
                        value={inputs.firstDayAllContracts} 
                        onChange={e => handleInputChange('firstDayAllContracts', e.target.value)} 
                    />
                </InputGroup>
                {getLSPDuration() && (
                    <div className="mt-2 text-gray-600 bg-gray-100 p-2 rounded">
                        服務年期: <span className="font-bold">{getLSPDuration().diffYears.toFixed(2)} 年</span> (共 {getLSPDuration().diffDays} 日)
                        {getLSPDuration().diffYears < 5 && <span className="text-red-500 block text-sm ml-1">(少於5年，不獲長期服務金)</span>}
                    </div>
                )}
               </>
            )}
          </div>
          
          <InputGroup label="其他款項 (如有) 銀碼如是減數就填寫，例如：-100">
             <input type="number" className="input-field" value={inputs.otherPayment} onChange={e => handleInputChange('otherPayment', e.target.value)} />
          </InputGroup>

          <InputGroup label="備註 (如有)">
             <input type="text" className="input-field" value={inputs.calcRemark} onChange={e => handleInputChange('calcRemark', e.target.value)} />
          </InputGroup>
        </div>

        <div className="mt-8">
          <button 
            onClick={handleCalculate}
            className="w-full bg-blue-600 hover:bg-blue-700 text-white font-bold py-4 px-6 rounded-lg transition-all flex items-center justify-center gap-2 shadow-lg text-xl"
          >
            <Calculator size={24}/> 計算銀碼
          </button>
        </div>
      </div>

      {/* Results & Part 2 Section */}
      {step >= 2 && editResults && (
        <div id="result-section" className="animate-fade-in">
          
          {/* Result Preview (Receipt) */}
          <div className="bg-gray-100 p-6 rounded-xl mb-8 border border-gray-300 overflow-auto">
            <h3 className="font-bold text-xl mb-4 text-gray-800 flex justify-between items-center bg-yellow-200 p-2 rounded">
                預覽計算結果 (可以直接點擊金額修改)：
                <span className="text-sm font-normal text-gray-600 flex items-center gap-1"><Edit2 size={16} className="inline"/>編輯模式</span>
            </h3>
            
            {/* The PDF Container - Using Flexbox to distribute space vertically */}
            <div id="pdf-content" ref={resultRef} className="bg-white p-8 shadow-lg text-gray-900 leading-relaxed mx-auto pdf-container flex flex-col justify-between" style={{ minHeight: '297mm', width: '210mm' }}>
              
              <div className="flex-grow flex flex-col">
                {/* Header */}
                <div className="text-center mb-1 mt-0">
                    <h1 className="text-xl font-bold tracking-wide border-b-2 border-gray-800 pb-1 inline-block">
                    在僱傭合約終止／屆滿時所付款項的收據<br/>
                    <span className="text-base">Receipt for Payments Upon Termination／<br/>Expiry of Employment Contract</span>
                    </h1>
                </div>

                {/* Personal Details */}
                <div className="grid grid-cols-1 gap-y-0.5 mb-2 text-base flex-grow-0">
                    <div className="flex border-b border-gray-300 pb-0.5">
                    <span className="w-1/2 font-bold">僱主姓名 Employer Name:</span>
                    <span className="w-1/2">{inputs.employerName}</span>
                    </div>
                    <div className="flex border-b border-gray-300 pb-0.5">
                    <span className="w-1/2 font-bold">外傭姓名 FDH Name:</span>
                    <span className="w-1/2">{inputs.fdhName}</span>
                    </div>
                    <div className="flex border-b border-gray-300 pb-0.5">
                    <span className="w-1/2 font-bold">外傭香港身份證號碼／護照號碼<br/>FDH HKID No.／Passport No.:</span>
                    <span className="w-1/2 self-end">{inputs.fdhId}</span>
                    </div>
                    <div className="flex border-b border-gray-300 pb-0.5">
                    <span className="w-1/2 font-bold">合約號碼 Contract No.:</span>
                    <span className="w-1/2">{inputs.contractNo}</span>
                    </div>
                    <div className="flex border-b border-gray-300 pb-0.5">
                    <span className="w-1/2 font-bold">款項收妥日期 Payment Received Date:</span>
                    <span className="w-1/2">{inputs.paymentDate ? formatDate(parseDate(inputs.paymentDate)) : ''}</span>
                    </div>
                    <div className="flex border-b border-gray-300 pb-0.5">
                    <span className="w-1/2 font-bold">支付方式 Payment Method:</span>
                    {/* Conditional Rendering for Payment Method Logic */}
                    <span className={`w-1/2 capitalize payment-method-text`}>
                        {inputs.paymentMethod === 'bank transfer' ? '銀行轉賬 Bank Transfer' : (inputs.paymentMethod === 'cheque' ? '支票 By Cheque' : '現金 In Cash')}
                    </span>
                    </div>
                </div>

                {/* Financial Items - Flex Grow to take up available space evenly */}
                <div className="flex-grow flex flex-col justify-around text-base space-y-0.5">
                    {/* (1) Last Wages */}
                    <div className="flex justify-between items-start border-b border-gray-200 pb-0.5">
                    <div>
                        <span className="font-bold block">(1) 最後工資 Last wages</span>
                        <span className="text-sm text-gray-600">
                        (由 from {formatDate(metaResults.dFirstDayTermMonth)} 至 to {formatDate(metaResults.dLastWork)})
                        </span>
                    </div>
                    <EditableAmount value={editResults.finalSalary} onChange={(val) => handleManualChange('finalSalary', val)} forceOneDec />
                    </div>

                    {/* (2) Untaken Leave */}
                    <div className="flex justify-between items-start border-b border-gray-200 pb-0.5">
                    <div>
                        <span className="font-bold block">(2) 未放取假期薪酬 Untaken leave pay</span>
                        <span className="text-sm text-gray-600 block">
                            未放年假+假期:{metaResults.finalLeaveDays}日Untaken paid annual leave+Untaken holiday:{metaResults.finalLeaveDays} day
                        </span>
                    </div>
                    <EditableAmount value={editResults.untakenLeavePay} onChange={(val) => handleManualChange('untakenLeavePay', val)} forceOneDec />
                    </div>

                    {/* (3) Transport */}
                    <div className="flex justify-between items-center border-b border-gray-200 pb-0.5">
                    <span className="font-bold">(3) 交通津貼 Travelling allowance</span>
                    <EditableAmount value={editResults.transport} onChange={(val) => handleManualChange('transport', val)} forceOneDec />
                    </div>

                    {/* (4) Ticket */}
                    <div className="flex justify-between items-center border-b border-gray-200 pb-0.5">
                    <span className="font-bold">(4) 回國機票款項 Payment in lieu of return air ticket</span>
                    <EditableAmount 
                        value={editResults.ticketCost} 
                        onChange={(val) => handleManualChange('ticketCost', val)} 
                        isText={isNaN(parseFloat(editResults.ticketCost))}
                    />
                    </div>

                    {/* (5) Notice Pay */}
                    <div className="flex justify-between items-start border-b border-gray-200 pb-0.5">
                    <div>
                        <span className="font-bold block">(5) 代通知金（如有）Payment in lieu of notice (if any)</span>
                        {inputs.hasNoticePay && (
                        <span className="text-sm text-gray-600">
                            通知日期 informed on {formatDate(parseDate(inputs.noticeDate))}
                        </span>
                        )}
                    </div>
                    <EditableAmount value={editResults.noticePay} onChange={(val) => handleManualChange('noticePay', val)} forceOneDec />
                    </div>

                    {/* (6) Long Service */}
                    <div className="flex justify-between items-center border-b border-gray-200 pb-0.5">
                    <span className="font-bold">(6) 長期服務金（如有）Long service payment (if any)</span>
                    <EditableAmount value={editResults.longServicePay} onChange={(val) => handleManualChange('longServicePay', val)} forceOneDec />
                    </div>

                    {/* (7) Other */}
                    <div className="flex justify-between items-center border-b border-gray-200 pb-0.5">
                    <span className="font-bold">(7) 其他 (如有) Other (if any)</span>
                    <EditableAmount value={editResults.other} onChange={(val) => handleManualChange('other', val)} forceOneDec />
                    </div>

                    {/* (8) Total */}
                    <div className="flex justify-between items-center pt-1 border-t-2 border-gray-800 mt-1">
                    <span className="text-lg font-bold uppercase">(8) 合共金額 TOTAL AMOUNT</span>
                    <span className="text-xl font-bold">HK${calculateTotal()}</span>
                    </div>

                    {/* (9) Remark */}
                    <div className="pt-1">
                    <span className="font-bold block">(9) 備註 (如有) Remark</span>
                    <div className="text-sm border-b border-gray-300 min-h-[1.2em] pb-0.5 mt-0.5 whitespace-pre-wrap">
                        {inputs.calcRemark}
                        {inputs.remark && (inputs.calcRemark ? "\n" : "") + inputs.remark}
                    </div>
                    </div>
                </div>

                {/* Declaration - Compact */}
                <div className="mb-1 text-[10px] text-justify leading-tight p-1 declaration-box flex-grow-0 mt-2">
                    <p className="mb-0.5 font-serif">
                    <strong>Declaration:</strong> We (the employer and the FDH) have agreed to and accepted all payments, holidays, and air-ticket arrangements. We will not make any form of complaint to the Philippine Consulate, Indonesian Consulate, Labour Department, Immigration Department, police station, and/or other government departments, nor will we seek any further losses, responsibilities, and/or compensation from the Hong Kong agency now or in the future.
                    </p>
                    <p className="font-serif">
                    <strong>聲明：</strong>我們（僱主及外傭）已同意及接受所有款項、假期和機票安排，並且我們不會向菲律賓領事館、印尼領事館、勞工處、入境處、警署及／或其他政府部門以作任何形式的投訴香港代理及／或日後追討其他損失、責任及／或款項。
                    </p>
                </div>
              </div>

              {/* Bottom Section: Signature Instructions and Signatures */}
              <div className="mt-auto">
                  {/* Signature Instruction Box */}
                  <div className="border border-black p-2 text-center text-xs font-bold mb-12 mt-2">
                      (簽名式樣必須與合約簽名相同。Signature should agree with that on the contract.)
                  </div>

                  {/* Signatures */}
                  <div className="flex justify-between items-end pb-0 signatures">
                    <div className="text-center w-5/12">
                      <div className="border-b border-black mb-1"></div>
                      <div className="font-bold text-sm">僱主簽署 Signature of Employer</div>
                    </div>
                    <div className="text-center w-5/12">
                      <div className="border-b border-black mb-1"></div>
                      <div className="font-bold text-sm">外傭簽署 Signature of FDH</div>
                    </div>
                  </div>
              </div>

            </div>
          </div>

          {/* Action Buttons */}
          <div className="mt-6 flex gap-4 flex-wrap">
               <button 
                 onClick={() => generatePDF(true)}
                 className="flex-1 bg-gray-600 hover:bg-gray-700 text-white font-bold py-4 px-6 rounded-lg shadow-lg text-lg flex items-center justify-center gap-2"
               >
                 <Download size={24} /> 輸出 PDF (沒有合約資料)
               </button>
               {step === 2 && (
                   <button 
                     onClick={handleShowContractForm}
                     className="flex-1 bg-green-600 hover:bg-green-700 text-white font-bold py-4 px-6 rounded-lg shadow-lg text-lg flex items-center justify-center gap-2"
                   >
                     填寫合約資料 <ChevronRight size={24} />
                   </button>
               )}
          </div>

          {/* Part 2: Contract Data Inputs */}
          {step === 3 && (
            <div ref={contractSectionRef} className="bg-white p-8 rounded-xl shadow-sm border border-gray-200 mt-8 animate-fade-in">
               <h2 className="text-2xl font-bold text-gray-800 border-b pb-4 mb-6 flex items-center gap-2">
                 <FileText className="text-green-600"/> 填寫合約資料
               </h2>
               
               {/* Validation Banner for Contract */}
               {generalError && (
                   <div className="mb-6 bg-red-100 border-l-4 border-red-500 text-red-700 p-4 rounded flex items-center gap-2 animate-fade-in">
                       <AlertCircle size={24} />
                       <p className="font-bold">{generalError}</p>
                   </div>
               )}

               <div className="grid grid-cols-1 gap-6">
                 <InputGroup label="僱主姓名 Employer Name" error={formErrors.employerName}>
                    <input type="text" className={`input-field ${formErrors.employerName ? 'border-red-500 bg-red-50' : ''}`} value={inputs.employerName} onChange={e => handleInputChange('employerName', e.target.value)} />
                 </InputGroup>
                 <InputGroup label="外傭姓名 FDH Name" error={formErrors.fdhName}>
                    <input type="text" className={`input-field ${formErrors.fdhName ? 'border-red-500 bg-red-50' : ''}`} value={inputs.fdhName} onChange={e => handleInputChange('fdhName', e.target.value)} />
                 </InputGroup>
                 <InputGroup label="外傭香港身份證號碼／護照號碼 FDH HKID No.／Passport No." error={formErrors.fdhId}>
                    <input type="text" className={`input-field ${formErrors.fdhId ? 'border-red-500 bg-red-50' : ''}`} value={inputs.fdhId} onChange={e => handleInputChange('fdhId', e.target.value)} />
                 </InputGroup>
                 <InputGroup label="合約號碼 Contract No." error={formErrors.contractNo}>
                    <input type="text" className={`input-field ${formErrors.contractNo ? 'border-red-500 bg-red-50' : ''}`} value={inputs.contractNo} onChange={e => handleInputChange('contractNo', e.target.value)} />
                 </InputGroup>
                 <InputGroup label="款項收妥日期 Payment Received Date" error={formErrors.paymentDate}>
                    <input type="date" className={`input-field ${formErrors.paymentDate ? 'border-red-500 bg-red-50' : ''}`} value={inputs.paymentDate} onChange={e => handleInputChange('paymentDate', e.target.value)} />
                 </InputGroup>
                 <InputGroup label="支付方式 Payment Method">
                    <select className="input-field" value={inputs.paymentMethod} onChange={e => handleInputChange('paymentMethod', e.target.value)}>
                      <option value="cash">現金 In Cash</option>
                      <option value="cheque">支票 By Cheque</option>
                      <option value="bank transfer">銀行轉賬 Bank Transfer</option>
                    </select>
                 </InputGroup>
                 <InputGroup label="備註 (如有)">
                     <textarea placeholder="非必要填寫，如要填寫備註，請以英文填寫" className="input-field" rows="3" value={inputs.remark} onChange={e => handleInputChange('remark', e.target.value)} />
                 </InputGroup>
               </div>

               <div className="mt-8">
                 <button 
                    onClick={() => generatePDF(false)}
                    disabled={!pdfLoaded}
                    className={`w-full bg-red-600 hover:bg-red-700 text-white font-bold py-4 px-6 rounded-lg shadow-lg flex items-center justify-center gap-2 text-xl ${!pdfLoaded ? 'opacity-50 cursor-not-allowed' : ''}`}
                 >
                   {pdfLoaded ? <><Download size={24} /> 輸出 PDF (有合約資料)</> : '載入 PDF 模組中...'}
                 </button>
               </div>
            </div>
          )}

        </div>
      )}
    </div>
  );
};

// Component for Editable Amounts - Uses Text input to allow comma formatting editing
const EditableAmount = ({ value, onChange, isText, forceOneDec }) => {
    let displayValue = value;
    if (!isText && value !== undefined && value !== '') {
        if (forceOneDec) {
            displayValue = formatMoney1DecStrict(value).replace('HK$', '');
        } else {
            displayValue = formatMoney1DecStrict(value).replace('HK$', '');
        }
    }
    
    return (
        <div className="text-right">
            {isText ? (
                <span className="font-bold text-gray-900">{value}</span>
            ) : (
                <div className="flex items-center justify-end">
                    <span className="text-gray-500 font-bold mr-1">HK$</span>
                     <input 
                        type="text" 
                        value={value} 
                        onChange={(e) => onChange(e.target.value)}
                        className="font-bold text-gray-900 w-32 text-right border-b border-dashed border-gray-400 focus:outline-none focus:border-blue-500 bg-transparent p-0 m-0"
                    />
                </div>
            )}
        </div>
    );
};

// --- 2. 4/6/8週計算機 (Visa Calculator) ---
const VisaCalculator = () => {
  const [date, setDate] = useState('');
  
  const calculateDate = (daysToSubtract) => {
    if (!date) return '-';
    const d = parseDate(date);
    d.setDate(d.getDate() - daysToSubtract);
    return formatDate(d);
  };

  return (
    <div className="bg-white p-8 rounded-xl shadow-sm border border-gray-100 max-w-2xl mx-auto">
      <h2 className="text-2xl font-bold text-gray-800 border-b pb-4 mb-6 flex items-center gap-2">
        <Calendar className="text-blue-600"/> 4/6/8週計算機
      </h2>
      
      <div className="mb-8">
        <label className="block text-base font-medium text-gray-700 mb-2">簽證完約日期 或 死移財最後工作日期</label>
        <input 
          type="date" 
          className="w-full p-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:outline-none text-lg"
          value={date} 
          onChange={e => setDate(e.target.value)} 
        />
      </div>

      <div className="space-y-4">
        <div className="text-sm text-gray-600 font-bold mb-2 bg-yellow-100 p-2 rounded">如果是印尼外傭，請在「最早遞交入境處」前2星期就開始驗身和遞交文件給印尼領事館</div>
        <VisaResultCard 
           title="完約轉換僱主 或 死移財轉換僱主 (4週)" 
           desc="簽證完約日期 - 28日" 
           date={calculateDate(28)} 
           color="bg-blue-50 text-blue-700 border-blue-200"
        />
        <VisaResultCard 
           title="Early Release (6週)" 
           desc="簽證完約日期 - 42日" 
           date={calculateDate(42)} 
           color="bg-orange-50 text-orange-700 border-orange-200"
        />
        <VisaResultCard 
           title="續約 (8週)" 
           desc="簽證完約日期 - 56日" 
           date={calculateDate(56)} 
           color="bg-green-50 text-green-700 border-green-200"
        />
      </div>
    </div>
  );
};

const VisaResultCard = ({ title, desc, date, color }) => (
  <div className={`p-6 rounded-lg border flex justify-between items-center ${color}`}>
    <div>
      <h3 className="font-bold text-lg">{title}</h3>
      <p className="text-sm opacity-75">{desc}</p>
    </div>
    <div className="text-right">
        <div className="text-2xl font-mono font-bold">{date}</div>
        <div className="text-xs opacity-75 mt-1">最早遞交入境處</div>
    </div>
  </div>
);

// --- 3. 放假計數機 (Leave Calculator) ---
const LeaveCalculator = () => {
  const [fwdStart, setFwdStart] = useState('');
  const [fwdDays, setFwdDays] = useState('');
  const [fwdExtras, setFwdExtras] = useState('');
  
  const [bwdEnd, setBwdEnd] = useState('');
  const [bwdDays, setBwdDays] = useState('');
  const [bwdExtras, setBwdExtras] = useState('');

  const [totalStart, setTotalStart] = useState('');
  const [totalEnd, setTotalEnd] = useState('');

  const getFwdSubtotal = () => {
      if (!fwdStart || !fwdDays) return null;
      const d = parseDate(fwdStart);
      d.setDate(d.getDate() + (parseInt(fwdDays) - 1));
      return d;
  };
  const fwdSubtotal = getFwdSubtotal();

  const calcForward = () => {
    if (!fwdSubtotal || fwdExtras === '') return '-';
    const d = new Date(fwdSubtotal);
    d.setDate(d.getDate() + parseInt(fwdExtras || 0));
    return formatDate(d);
  };

  const getFwdSpan = () => {
      const endStr = calcForward();
      if (endStr === '-') return '-';
      const start = parseDate(fwdStart);
      const sub = getFwdSubtotal();
      const end = new Date(sub);
      end.setDate(end.getDate() + parseInt(fwdExtras || 0));
      
      const diff = Math.ceil((end - start) / MS_PER_DAY) + 1;
      return diff + " 日";
  };


  const getBwdSubtotal = () => {
     if (!bwdEnd || !bwdDays) return null;
     const d = parseDate(bwdEnd);
     d.setDate(d.getDate() - parseInt(bwdDays) + 1);
     return d;
  };
  const bwdSubtotal = getBwdSubtotal();

  const calcBackward = () => {
    if (!bwdSubtotal || bwdExtras === '') return '-';
    const d = new Date(bwdSubtotal);
    d.setDate(d.getDate() - parseInt(bwdExtras || 0));
    return formatDate(d);
  };

  const getBwdSpan = () => {
      const startStr = calcBackward();
      if (startStr === '-') return '-';
      const end = parseDate(bwdEnd);
      const sub = getBwdSubtotal();
      const start = new Date(sub);
      start.setDate(start.getDate() - parseInt(bwdExtras || 0));

      const diff = Math.ceil((end - start) / MS_PER_DAY) + 1;
      return diff + " 日";
  };

  const calcTotal = () => {
    if (!totalStart || !totalEnd) return '-';
    const s = parseDate(totalStart);
    const e = parseDate(totalEnd);
    const diff = Math.ceil((e - s) / MS_PER_DAY) + 1;
    return `${diff} 日`;
  };

  return (
    <div className="grid gap-8 md:grid-cols-1 lg:grid-cols-1">
      
      {/* 順計日子 */}
      <div className="bg-white p-8 rounded-xl shadow-sm border border-gray-200">
        <h3 className="font-bold text-xl text-blue-600 mb-6 border-b pb-2 flex items-center gap-2"><CalendarCheck size={24} /> <ArrowRight size={24} /> 順計日子 (已有放假第一天，要計算最後一天)</h3>
        <div className="space-y-5">
          <InputGroup label="放假第一天 (A)">
            <input type="date" className="input-field" value={fwdStart} onChange={e => setFwdStart(e.target.value)} />
          </InputGroup>
          
          <InputGroup label="放幾多日有薪年假">
            <input type="number" className="input-field" value={fwdDays} onChange={e => setFwdDays(e.target.value)} />
            <div className="mt-2 text-blue-800 bg-blue-100 border border-blue-200 p-2 rounded text-lg">
                放假最後一天 (B) (小計)： <span className="font-bold">{fwdSubtotal ? formatDate(fwdSubtotal) : '-'}</span>
            </div>
          </InputGroup>

          <InputGroup label="(A) 至 (B) 之間有多少個 休息日 和 勞工假期">
            <input type="number" className="input-field" value={fwdExtras} onChange={e => setFwdExtras(e.target.value)} />
          </InputGroup>
          
          <div className="grid grid-cols-2 gap-4 mt-4">
             <div className="p-4 bg-blue-50 text-blue-800 rounded flex flex-col justify-center items-center h-32">
                <span className="text-sm block text-gray-500 mb-1">放假最後一天 (最終)</span>
                <span className="text-2xl font-bold">{calcForward()}</span>
             </div>
             <div className="p-4 bg-blue-50 text-blue-800 rounded flex flex-col justify-center items-center h-32">
                <span className="text-sm block text-gray-500 mb-1">橫跨日數</span>
                <span className="text-xl font-bold">{getFwdSpan()}</span>
             </div>
          </div>
        </div>
      </div>

      {/* 倒計日子 */}
      <div className="bg-white p-8 rounded-xl shadow-sm border border-gray-200">
        <h3 className="font-bold text-xl text-orange-600 mb-6 border-b pb-2 flex items-center gap-2"><CalendarCheck size={24} /> <ArrowLeft size={24} /> 倒計日子 (已有放假最後一天，要計算第一天)</h3>
        <div className="space-y-5">
          <InputGroup label="放假最後一天 (A)">
            <input type="date" className="input-field" value={bwdEnd} onChange={e => setBwdEnd(e.target.value)} />
          </InputGroup>
          
          <InputGroup label="放幾多日有薪年假">
            <input type="number" className="input-field" value={bwdDays} onChange={e => setBwdDays(e.target.value)} />
            <div className="mt-2 text-orange-800 bg-orange-100 border border-orange-200 p-2 rounded text-lg">
                放假第一天 (B) (小計)： <span className="font-bold">{bwdSubtotal ? formatDate(bwdSubtotal) : '-'}</span>
            </div>
          </InputGroup>
          
          <InputGroup label="(A) 至 (B) 之間有多少個 休息日 和 勞工假期">
            <input type="number" className="input-field" value={bwdExtras} onChange={e => setBwdExtras(e.target.value)} />
          </InputGroup>
          
          <div className="grid grid-cols-2 gap-4 mt-4">
             <div className="p-4 bg-orange-50 text-orange-800 rounded flex flex-col justify-center items-center h-32">
                <span className="text-sm block text-gray-500 mb-1">放假第一天 (最終)</span>
                <span className="text-2xl font-bold">{calcBackward()}</span>
             </div>
             <div className="p-4 bg-orange-50 text-orange-800 rounded flex flex-col justify-center items-center h-32">
                <span className="text-sm block text-gray-500 mb-1">橫跨日數</span>
                <span className="text-xl font-bold">{getBwdSpan()}</span>
             </div>
          </div>
        </div>
      </div>

      {/* 總日子 */}
      <div className="bg-white p-8 rounded-xl shadow-sm border border-gray-200">
        <h3 className="font-bold text-xl text-green-600 mb-6 border-b pb-2 flex items-center gap-2"><CalendarCheck size={24} /> <ArrowRightLeft size={24} /> 固定放假日子計算 (不計算當中休息日 和 勞工假期)</h3>
        <div className="space-y-5">
          <InputGroup label="放假第一天 (最終)">
            <input type="date" className="input-field" value={totalStart} onChange={e => setTotalStart(e.target.value)} />
          </InputGroup>
          <InputGroup label="放假最後一天 (最終)">
            <input type="date" className="input-field" value={totalEnd} onChange={e => setTotalEnd(e.target.value)} />
          </InputGroup>
          <div className="mt-8 p-4 bg-green-50 text-green-800 rounded text-center">
            <span className="text-sm block text-gray-500">橫跨日數 (如果當中有休息日 和 勞工假期，即外傭實際放取有薪年假應少過橫跨日數)</span>
            <span className="text-3xl font-bold">{calcTotal()}</span>
          </div>
        </div>
      </div>
    </div>
  );
};

const ChevronRight = ({ size }) => (
    <svg xmlns="http://www.w3.org/2000/svg" width={size} height={size} viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="m9 18 6-6-6-6"/></svg>
);

const InputGroup = ({ label, children, error }) => (
  <div className="w-full">
    <label className={`block text-base font-bold mb-2 ${error ? 'text-red-600' : 'text-gray-700'}`}>{label}</label>
    {children}
  </div>
);

// Styles including PDF specific compression and hiding logic
const styles = `
  .input-field {
    width: 100%;
    padding: 0.75rem 1rem;
    border: 1px solid #9ca3af;
    border-radius: 0.5rem;
    font-size: 1.125rem;
    transition: all 0.2s;
  }
  .input-field:focus {
    outline: none;
    border-color: #2563eb;
    box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.1);
  }
  .no-scrollbar::-webkit-scrollbar {
    display: none;
  }
  .no-scrollbar {
    -ms-overflow-style: none;
    scrollbar-width: none;
  }
  @keyframes fade-in {
    from { opacity: 0; transform: translateY(10px); }
    to { opacity: 1; transform: translateY(0); }
  }
  .animate-fade-in {
    animation: fade-in 0.3s ease-out forwards;
  }

  /* PDF Generation Styles */
  .pdf-mode {
      transform-origin: top left;
      width: 210mm !important;
      min-height: 297mm !important;
      padding: 15mm 15mm 10mm 15mm !important; /* Adjusted Padding */
      display: flex !important;
      flex-direction: column !important;
      justify-content: space-between !important;
  }
  /* Hiding Payment Info specifically for No-Contract PDF */
  .pdf-mode.no-contract-mode .payment-method-text {
      visibility: hidden;
  }

  .pdf-mode h1 {
      font-size: 16pt !important;
      margin-bottom: 2mm !important;
      margin-top: 0 !important;
  }
  .pdf-mode span {
      font-size: 10pt !important;
  }
  .pdf-mode .font-bold {
      font-weight: 700 !important;
  }
  .pdf-mode .grid {
      gap: 0px !important;
      margin-bottom: 2mm !important;
  }
  .pdf-mode .space-y-0.5 > * {
      padding-bottom: 0px !important;
      margin-bottom: 1px !important;
      line-height: 1.15 !important;
  }
  .pdf-mode .text-2xl {
      font-size: 14pt !important;
  }
  .pdf-mode .text-xl {
      font-size: 12pt !important;
  }
  .pdf-mode .declaration-box {
      margin-top: 2mm !important;
      font-size: 8pt !important;
      padding: 1mm !important;
      line-height: 1.1 !important;
  }
  .pdf-mode .signatures {
      margin-top: auto !important;
      padding-top: 0 !important;
      padding-bottom: 0 !important;
  }
  .pdf-mode .pdf-container {
      transform: scale(0.98); 
      transform-origin: top center;
  }
`;

const StyleInjector = () => (
  <style>{styles}</style>
);

export default App;
