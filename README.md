 Apps.js
document.addEventListener('DOMContentLoaded', () => {

    const saveBtn = document.getElementById('saveBtn');
    const newBtn = document.getElementById('newBtn');
    const searchInput = document.getElementById('searchInput');
    const licensesTableBody = document.querySelector('#licensesTable tbody');
    const buildingTypeSelect = document.getElementById('buildingType');
    const floorGroup = document.getElementById('floor-group');
    const exportBtn = document.getElementById('exportData');
    const importBtn = document.getElementById('importDataBtn');
    const importFile = document.getElementById('importFile');
    const printAllBtn = document.getElementById('printAll');
    const form = document.getElementById('license-form').querySelector('form');

    let licenses = JSON.parse(localStorage.getItem('licensesData_v1')) || [];
    let currentlyEditingIndex = null;
    let fileAttachments = [];

    const renderTable = () => {
        licensesTableBody.innerHTML = '';
        const searchTerm = searchInput.value.toLowerCase();
        const filteredLicenses = licenses.filter(license => license.licenseId.toLowerCase().startsWith(searchTerm));

        if (filteredLicenses.length === 0) {
            licensesTableBody.innerHTML = `<tr><td colspan="5" style="text-align:center; padding: 20px;">لا توجد بيانات لعرضها</td></tr>`;
            return;
        }

        filteredLicenses.forEach(license => {
            const originalIndex = licenses.findIndex(l => l.id === license.id);
            const tr = document.createElement('tr');
            tr.innerHTML = `
                <td>${escapeHTML(license.licenseId)}</td>
                <td>${escapeHTML(license.applicantName)}</td>
                <td>${escapeHTML(license.location)}</td>
                <td>${escapeHTML(license.issueDate)}</td>
                <td class="actions-cell">
                    <button class="btn-warning" onclick="editLicense(${originalIndex})">تعديل</button>
                    <button class="btn-info" onclick="printLicense(${originalIndex})">طباعة</button>
                    <button class="btn-danger" onclick="deleteLicense(${originalIndex})">حذف</button>
                </td>
            `;
            licensesTableBody.appendChild(tr);
        });
    };
    
    const saveData = () => {
        localStorage.setItem('licensesData_v1', JSON.stringify(licenses));
    };

    const clearForm = () => {
        form.reset();
        currentlyEditingIndex = null;
        fileAttachments = [];
        document.getElementById('attachments').value = '';
        saveBtn.textContent = '💾 حفظ';
        saveBtn.classList.remove('btn-warning');
        saveBtn.classList.add('btn-success');
        buildingTypeSelect.dispatchEvent(new Event('change'));
    };
    
    const handleFileUpload = (files) => {
        return Promise.all(Array.from(files).map(file => new Promise((resolve, reject) => {
            const reader = new FileReader();
            reader.onload = e => resolve({ name: file.name, type: file.type, data: e.target.result });
            reader.onerror = e => reject(e);
            reader.readAsDataURL(file);
        })));
    };
    
    const escapeHTML = str => str.toString().replace(/[&<>"']/g, m => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' })[m]);

    saveBtn.addEventListener('click', async () => {
        const licenseIdInput = document.getElementById('licenseId');
        if (!licenseIdInput.value.trim()) {
            alert('يرجى إدخال رقم الرخصة.');
            licenseIdInput.focus();
            return;
        }

        const attachmentInput = document.getElementById('attachments');
        if (attachmentInput.files.length > 0) {
            try {
                fileAttachments = await handleFileUpload(attachmentInput.files);
            } catch (error) {
                alert("حدث خطأ أثناء تحميل الملفات.");
                return;
            }
        }
       
        const licenseData = {
            id: currentlyEditingIndex !== null ? licenses[currentlyEditingIndex].id : Date.now(),
            licenseId: document.getElementById('licenseId').value,
            licenseType: document.querySelector('input[name="licenseType"]:checked').value,
            buildingType: document.getElementById('buildingType').value,
            floorLevel: floorGroup.style.display !== 'none' ? document.getElementById('floorLevel').value : null,
            location: document.getElementById('location').value,
            applicantName: document.getElementById('applicantName').value,
            applicantId: document.getElementById('applicantId').value,
            applicantPhone: document.getElementById('applicantPhone').value,
            borders: { north: document.getElementById('borderNorth').value, south: document.getElementById('borderSouth').value, east: document.getElementById('borderEast').value, west: document.getElementById('borderWest').value },
            dimensions: { north: document.getElementById('dimNorth').value, south: document.getElementById('dimSouth').value, east: document.getElementById('dimEast').value, west: document.getElementById('dimWest').value },
            area: document.getElementById('area').value,
            issueDate: document.getElementById('issueDate').value,
            fees: {
                build: { voucher: document.getElementById('feeBuildVoucher').value, date: document.getElementById('feeBuildDate').value, amount: document.getElementById('feeBuildAmount').value },
                plan: { voucher: document.getElementById('feePlanVoucher').value, date: document.getElementById('feePlanDate').value, amount: document.getElementById('feePlanAmount').value },
                waste: { voucher: document.getElementById('feeWasteVoucher').value, date: document.getElementById('feeWasteDate').value, amount: document.getElementById('feeWasteAmount').value },
                improve: { voucher: document.getElementById('feeImproveVoucher').value, date: document.getElementById('feeImproveDate').value, amount: document.getElementById('feeImproveAmount').value }
            },
            attachments: fileAttachments
        };

        if (currentlyEditingIndex !== null) {
            if(fileAttachments.length === 0 && licenses[currentlyEditingIndex].attachments) {
                licenseData.attachments = licenses[currentlyEditingIndex].attachments;
            }
            licenses[currentlyEditingIndex] = licenseData;
        } else {
            licenses.push(licenseData);
        }

        saveData();
        renderTable();
        clearForm();
        alert('تم حفظ البيانات بنجاح!');
    });

    newBtn.addEventListener('click', clearForm);
    searchInput.addEventListener('input', renderTable);
    
    buildingTypeSelect.addEventListener('change', () => {
        floorGroup.style.display = ['مبنى سكني', 'مبنى تجاري'].includes(buildingTypeSelect.value) ? 'block' : 'none';
    });

    exportBtn.addEventListener('click', () => {
        if (licenses.length === 0) return alert('لا توجد بيانات لتصديرها.');
        const dataStr = JSON.stringify(licenses, null, 2);
        const dataBlob = new Blob([dataStr], { type: 'application/json' });
        const url = URL.createObjectURL(dataBlob);
        const a = document.createElement('a');
        const timestamp = new Date().toISOString().slice(0, 19).replace(/:/g, '-');
        a.href = url;
        a.download = `تصدير_بيانات_التراخيص_${timestamp}.json`;
        a.click();
        URL.revokeObjectURL(url);
    });
    
    importBtn.addEventListener('click', () => importFile.click());
    importFile.addEventListener('change', (event) => {
        const file = event.target.files[0];
        if (!file) return;
        const reader = new FileReader();
        reader.onload = (e) => {
            try {
                const importedData = JSON.parse(e.target.result);
                if (!Array.isArray(importedData)) throw new Error("Invalid format");
                
                const choice = confirm("اختر طريقة الاستيراد:\n\n- اضغط 'موافق' (OK) لإضافة البيانات للبيانات الحالية.\n- اضغط 'إلغاء' (Cancel) لحذف البيانات الحالية واستبدالها بالجديدة.");

                if (choice) { licenses = [...licenses, ...importedData.filter(imp => !licenses.some(exi => exi.id === imp.id))] } 
                else { licenses = importedData; }
                
                saveData();
                renderTable();
                alert('تم استيراد البيانات بنجاح!');
            } catch (error) {
                alert('خطأ في استيراد الملف. تأكد من أن الملف بصيغة JSON صحيحة.');
            } finally {
                importFile.value = '';
            }
        };
        reader.readAsText(file);
    });

    printAllBtn.addEventListener('click', () => {
        const printWindow = window.open('', '_blank');
        printWindow.document.write(`<html><head><title>طباعة كل التراخيص</title><link rel="stylesheet" href="style.css"></head><body>`);
        printWindow.document.write('<header><h1>مكتب الأشغال العامة والطرق</h1><p>قسم تراخيص البناء</p></header>');
        printWindow.document.write(document.querySelector('.table-container').innerHTML);
        printWindow.document.write('</body></html>');
        printWindow.document.close();
        setTimeout(() => { // Wait for content to load
            printWindow.print();
            printWindow.close();
        }, 250);
    });

    window.editLicense = (index) => {
        clearForm();
        currentlyEditingIndex = index;
        const license = licenses[index];
        Object.keys(license).forEach(key => {
            if(document.getElementById(key)) document.getElementById(key).value = license[key];
        });
        document.querySelector(`input[name="licenseType"][value="${license.licenseType}"]`).checked = true;
        if(license.floorLevel) document.getElementById('floorLevel').value = license.floorLevel;
        Object.keys(license.borders).forEach(key => document.getElementById('border' + key.charAt(0).toUpperCase() + key.slice(1)).value = license.borders[key]);
        Object.keys(license.dimensions).forEach(key => document.getElementById('dim' + key.charAt(0).toUpperCase() + key.slice(1)).value = license.dimensions[key]);
        Object.keys(license.fees).forEach(type => Object.keys(license.fees[type]).forEach(prop => document.getElementById('fee' + type.charAt(0).toUpperCase() + type.slice(1) + prop.charAt(0).toUpperCase() + prop.slice(1)).value = license.fees[type][prop]));
        fileAttachments = license.attachments || [];
        saveBtn.textContent = '💾 تحديث';
        saveBtn.classList.replace('btn-success', 'btn-warning');
        window.scrollTo({ top: 0, behavior: 'smooth' });
    };

    window.deleteLicense = (index) => {
        if (confirm(`هل أنت متأكد من حذف الرخصة رقم ${licenses[index].licenseId}؟`)) {
            licenses.splice(index, 1);
            saveData();
            renderTable();
        }
    };

    window.printLicense = (index) => {
        const license = licenses[index];
        let attachmentsHtml = '';
        if (license.attachments && license.attachments.length > 0) {
            attachmentsHtml += '<h2>الملفات المرفقة</h2>';
            license.attachments.forEach(file => {
                attachmentsHtml += file.type.startsWith('image/')
                    ? `<img src="${file.data}" alt="${escapeHTML(file.name)}" style="max-width: 100%; margin-bottom: 20px; border: 1px solid #ccc;">`
                    : `<div class="print-pdf"><p>ملف PDF: ${escapeHTML(file.name)}</p><iframe src="${file.data}" width="100%" height="500px"></iframe></div>`;
            });
        }
        const printContent = `
            <h1>مكتب الأشغال العامة والطرق - قسم تراخيص البناء</h1>
            <h2>تفاصيل الرخصة رقم: ${escapeHTML(license.licenseId)}</h2>
            <table class="print-table">
                <tr><td>رقم الرخصة</td><td>${escapeHTML(license.licenseId)}</td></tr>
                <tr><td>نوع الرخصة</td><td>${escapeHTML(license.licenseType)}</td></tr>
                <tr><td>نوع المبنى</td><td>${escapeHTML(license.buildingType)} ${license.floorLevel ? `(${escapeHTML(license.floorLevel)})` : ''}</td></tr>
                <tr><td>تاريخ الإصدار</td><td>${escapeHTML(license.issueDate)}</td></tr>
                <tr><td>اسم طالب الرخصة</td><td>${escapeHTML(license.applicantName)}</td></tr>
                <tr><td>رقم البطاقة/الجواز</td><td>${escapeHTML(license.applicantId)}</td></tr>
                <tr><td>رقم الهاتف</td><td>${escapeHTML(license.applicantPhone)}</td></tr>
                <tr><td>موقع المبنى</td><td>${escapeHTML(license.location)}</td></tr>
                <tr><td>المساحة</td><td>${escapeHTML(license.area)} م²</td></tr>
                <tr><td>الحدود</td><td>ش: ${escapeHTML(license.borders.north)}, ج: ${escapeHTML(license.borders.south)}, ش: ${escapeHTML(license.borders.east)}, غ: ${escapeHTML(license.borders.west)}</td></tr>
                <tr><td>الأبعاد</td><td>ش: ${escapeHTML(license.dimensions.north)}م, ج: ${escapeHTML(license.dimensions.south)}م, ش: ${escapeHTML(license.dimensions.east)}م, غ: ${escapeHTML(license.dimensions.west)}م</td></tr>
            </table>
            <h2>تفاصيل الرسوم المالية</h2>
            <table class="print-table">
               <thead><tr><th>البيان</th><th>رقم السند</th><th>تاريخ السند</th><th>المبلغ</th></tr></thead>
               <tbody>
                  <tr><td>رسوم البناء</td><td>${escapeHTML(license.fees.build.voucher)}</td><td>${escapeHTML(license.fees.build.date)}</td><td>${escapeHTML(license.fees.build.amount)}</td></tr>
                  <tr><td>رسوم التخطيط</td><td>${escapeHTML(license.fees.plan.voucher)}</td><td>${escapeHTML(license.fees.plan.date)}</td><td>${escapeHTML(license.fees.plan.amount)}</td></tr>
                  <tr><td>رسوم رفع المخلفات</td><td>${escapeHTML(license.fees.waste.voucher)}</td><td>${escapeHTML(license.fees.waste.date)}</td><td>${escapeHTML(license.fees.waste.amount)}</td></tr>
                  <tr><td>رسوم التحسين</td><td>${escapeHTML(license.fees.improve.voucher)}</td><td>${escapeHTML(license.fees.improve.date)}</td><td>${escapeHTML(license.fees.improve.amount)}</td></tr>
               </tbody>
            </table>
            <div class="print-attachments">${attachmentsHtml}</div>
        `;
        
        const printWindow = window.open();
        printWindow.document.write(`<html><head><title>طباعة رخصة</title><link rel="stylesheet" href="style.css"></head><body>${printContent}</body></html>`);
        printWindow.document.close();
        printWindow.focus();
        setTimeout(() => { printWindow.print(); printWindow.close(); }, 500);
    };

    renderTable();
    buildingTypeSelect.dispatchEvent(new Event('change'));
});





Dashboard.html
<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>مكتب الأشغال العامة والطرق - قسم تراخيص البناء</title>
    <style>
        /* CSS مدمج ومحسن */
        @import url('https://fonts.googleapis.com/css2?family=Tajawal:wght@400;700&display=swap');
        :root {
            --primary-color: #4a69bd; --secondary-color: #6a89cc; --accent-color: #f6b93b;
            --background-start: #f7f7f7; --background-end: #e2e8f0; --surface-color: rgba(255, 255, 255, 0.95);
            --text-color: #1e272e; --danger-color: #e55039; --success-color: #38ada9;
            --border-radius: 12px; --shadow: 0 10px 30px rgba(0, 0, 0, 0.1);
        }
        * { box-sizing: border-box; margin: 0; padding: 0; }
        body { font-family: 'Tajawal', sans-serif; direction: rtl; background-image: linear-gradient(to top left, var(--background-start), var(--background-end)); color: var(--text-color); line-height: 1.7; padding: 20px; min-height: 100vh; }
        .container { max-width: 1200px; margin: auto; background: var(--surface-color); padding: 30px; border-radius: var(--border-radius); box-shadow: var(--shadow); border: 1px solid rgba(255, 255, 255, 0.2); }
        header { text-align: center; padding: 20px; background-image: linear-gradient(45deg, var(--primary-color), var(--secondary-color)); color: white; border-radius: var(--border-radius); margin-bottom: 25px; text-shadow: 1px 1px 3px rgba(0,0,0,0.2); }
        header h1 { margin: 0; font-size: 2.2rem; }
        header p { margin: 5px 0 0; font-size: 1.3rem; opacity: 0.9; }
        footer { text-align: center; margin-top: 30px; padding: 15px; color: var(--text-color); opacity: 0.7; }
        footer a { color: var(--primary-color); font-weight: bold; text-decoration: none;}
        .controls, .main-actions { display: flex; flex-wrap: wrap; gap: 15px; margin-bottom: 25px; align-items: center; }
        input, select, textarea { width: 100%; padding: 14px; border: 1px solid #ddd; border-radius: var(--border-radius); font-family: 'Tajawal', sans-serif; font-size: 1rem; transition: all 0.3s ease; background-color: #fcfcfc; }
        input:focus, select:focus, textarea:focus { outline: none; border-color: var(--primary-color); box-shadow: 0 0 0 3px rgba(74, 105, 189, 0.2); }
        button, .btn { padding: 14px 22px; border: none; border-radius: var(--border-radius); background-color: var(--primary-color); color: white; font-family: inherit; font-size: 1rem; font-weight: 700; cursor: pointer; transition: all 0.3s ease; box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1); display: inline-flex; align-items: center; justify-content: center; gap: 8px; }
        button:hover, .btn:hover { transform: translateY(-3px) scale(1.03); box-shadow: 0 8px 20px rgba(0, 0, 0, 0.15); }
        button:active { transform: translateY(-1px) scale(1); }
        button.btn-success { background-color: var(--success-color); }
        button.btn-danger { background-color: var(--danger-color); }
        button.btn-warning { background-color: var(--accent-color); color: #333; }
        button.btn-info { background-color: var(--secondary-color); }
        #form-toggle-container { padding: 15px; border-bottom: 1px solid #eee; margin-bottom: 20px; }
        #license-form { background: transparent; padding-top: 20px; border-top: 1px solid #eee; margin-top: 20px; display: none; /* مخفي افتراضيا */ }
        .form-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(280px, 1fr)); gap: 25px; }
        .form-group label { margin-bottom: 8px; font-weight: 700; font-size: 0.95rem; display: block; }
        #attachments-list { font-size: 0.9em; color: #555; margin-top: 10px; }
        .radio-group { display: flex; gap: 20px; align-items: center; padding-top: 10px; }
        fieldset { border: 1px solid #ddd; border-radius: var(--border-radius); padding: 20px; margin-top: 10px; transition: all 0.3s ease; }
        fieldset:hover { border-color: var(--secondary-color); }
        fieldset legend { padding: 0 10px; font-weight: 700; color: var(--primary-color); }
        .table-container { overflow-x: auto; border: 1px solid #e0e0e0; border-radius: var(--border-radius); box-shadow: 0 4px 10px rgba(0,0,0,0.05); }
        table { width: 100%; border-collapse: collapse; }
        th, td { padding: 16px; text-align: right; border-bottom: 1px solid #e0e0e0; }
        thead th { background-color: #f8f9fa; color: var(--text-color); font-size: 1.1rem; position: sticky; top: 0; z-index: 10; }
        tbody tr { transition: background-color 0.3s ease; }
        tbody tr:last-child td { border-bottom: none; }
        tbody tr:hover { background-color: rgba(106, 137, 204, 0.1); }
        .actions-cell button { padding: 8px 12px; margin: 0 4px; font-size: 0.9rem; }
        .backup-tip { background-color: #fffbe6; border: 1px solid #ffe58f; border-radius: var(--border-radius); padding: 15px; margin: 20px 0; font-size: 0.95rem; }
        #toast-notification { position: fixed; bottom: 20px; left: 50%; transform: translateX(-50%); background-color: var(--success-color); color: white; padding: 15px 30px; border-radius: var(--border-radius); box-shadow: var(--shadow); z-index: 2000; opacity: 0; visibility: hidden; transition: opacity 0.5s, visibility 0.5s, bottom 0.5s; }
        #toast-notification.show { opacity: 1; visibility: visible; bottom: 30px; }
    </style>
</head>
<body>
    <header>
        <h1>مكتب الأشغال العامة والطرق</h1>
        <p>قسم تراخيص البناء</p>
    </header>
    <div class="container">
        <div class="controls">
            <button id="exportData" class="btn-info">⤓ تصدير جميع البيانات</button>
            <button id="importDataBtn" class="btn-info">⤒ استيراد جميع البيانات</button>
            <input type="file" id="importFile" accept=".json" style="display: none;">
            <input type="text" id="searchInput" placeholder="🔎 ابحث برقم الرخصة..." style="flex-grow: 1;">
            <button id="printAll" class="btn-warning">📠 طباعة جدول التراخيص</button>
        </div>
        <div class="backup-tip">💡 <b>نصيحة هامة:</b> زر "تصدير البيانات" يقوم بإنشاء نسخة احتياطية دائمة على هاتفك. قم بالتصدير بشكل دوري لحماية بياناتك من الضياع.</div>
        <div id="form-toggle-container">
            <button id="showFormBtn" class="btn-success">➕ إضافة رخصة جديدة</button>
        </div>
        <div id="license-form">
            <form class="form-grid" onsubmit="return false;">
                <div class="form-group"><label for="licenseId">رقم الرخصة</label><input type="text" id="licenseId" required></div>
                <div class="form-group"><label>نوع الرخصة</label><div class="radio-group"><label><input type="radio" name="licenseType" value="جديدة" checked> جديدة</label><label><input type="radio" name="licenseType" value="تجديد"> تجديد</label></div></div>
                <div class="form-group"><label for="buildingType">نوع المبنى</label><select id="buildingType"><option value="مبنى سكني">مبنى سكني</option><option value="مبنى تجاري">مبنى تجاري</option><option value="سور">سور</option><option value="بيارة">بيارة</option></select></div>
                <div class="form-group" id="floor-group"><label for="floorLevel">الدور</label><select id="floorLevel"><option value="دور أرضي">دور أرضي</option><option value="الدور الأول">الدور الأول</option><option value="الدور الثاني">الدور الثاني</option><option value="الدور الثالث">الدور الثالث</option><option value="الدور الرابع">الدور الرابع</option><option value="الدور الخامس">الدور الخامس</option></select></div>
                <div class="form-group" style="grid-column: 1 / -1;"><label for="location">موقع المبنى</label><textarea id="location" rows="2"></textarea></div>
                <div class="form-group"><label for="applicantName">اسم طالب الرخصة</label><input type="text" id="applicantName"></div>
                <div class="form-group"><label for="applicantId">رقم البطاقة/الجواز</label><input type="text" id="applicantId"></div>
                <div class="form-group"><label for="applicantPhone">رقم الهاتف</label><input type="text" id="applicantPhone"></div>
                <fieldset style="grid-column: 1 / -1;"><legend>الحدود</legend><div class="form-grid"><input type="text" id="borderNorth" placeholder="الشمال"><input type="text" id="borderSouth" placeholder="الجنوب"><input type="text" id="borderEast" placeholder="الشرق"><input type="text" id="borderWest" placeholder="الغرب"></div></fieldset>
                <fieldset style="grid-column: 1 / -1;"><legend>الأبعاد (بالمتر)</legend><div class="form-grid"><input type="number" id="dimNorth" placeholder="الشمال"><input type="number" id="dimSouth" placeholder="الجنوب"><input type="number" id="dimEast" placeholder="الشرق"><input type="number" id="dimWest" placeholder="الغرب"></div></fieldset>
                <div class="form-group"><label for="area">المساحة (م²)</label><input type="number" id="area" placeholder="أدخل المساحة يدوياً"></div>
                <div class="form-group"><label for="issueDate">تاريخ إصدار الرخصة</label><input type="date" id="issueDate"></div>
                <div class="form-group" style="grid-column: span 2;"><label for="attachments">الملفات المرفقة (صور, PDF)</label><input type="file" id="attachments" multiple accept="image/*,application/pdf"><div id="attachments-list"></div></div>
                <fieldset style="grid-column: 1 / -1;"><legend>الرسوم المالية</legend><div class="form-grid" style="grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));"><fieldset><legend>رسوم البناء</legend><input type="text" id="feeBuildVoucher" placeholder="رقم السند"><input type="date" id="feeBuildDate" style="margin-top:5px;"><input type="number" id="feeBuildAmount" placeholder="قيمة السند" style="margin-top:5px;"></fieldset><fieldset><legend>رسوم التخطيط</legend><input type="text" id="feePlanVoucher" placeholder="رقم السند"><input type="date" id="feePlanDate" style="margin-top:5px;"><input type="number" id="feePlanAmount" placeholder="قيمة السند" style="margin-top:5px;"></fieldset><fieldset><legend>رسوم رفع المخلفات</legend><input type="text" id="feeWasteVoucher" placeholder="رقم السند"><input type="date" id="feeWasteDate" style="margin-top:5px;"><input type="number" id="feeWasteAmount" placeholder="قيمة السند" style="margin-top:5px;"></fieldset><fieldset><legend>رسوم التحسين</legend><input type="text" id="feeImproveVoucher" placeholder="رقم السند"><input type="date" id="feeImproveDate" style="margin-top:5px;"><input type="number" id="feeImproveAmount" placeholder="قيمة السند" style="margin-top:5px;"></fieldset></div></fieldset>
            </form>
            <div class="main-actions"><button id="saveBtn" class="btn-success">💾 حفظ</button><button id="cancelBtn" class="btn-danger">إلغاء</button></div>
        </div>
        <div class="table-container"><table id="licensesTable"><thead><tr><th>رقم الرخصة</th><th>اسم طالب الرخصة</th><th>الموقع</th><th>تاريخ الإصدار</th><th class="actions-cell">الأوامر</th></tr></thead><tbody></tbody></table></div>
    </div>
    <div id="toast-notification"></div>
    <footer>
         <p>إعداد: م. وليد غنيمة | <a href="tel:+967775044084">+967775044084</a> | <a href="mailto:wajag93@gmail.com">wajag93@gmail.com</a></p>
    </footer>
    <script>
        // JS مدمج ومُصحح بالكامل
        document.addEventListener('DOMContentLoaded', () => {
            const licenseForm = document.getElementById('license-form'); const showFormBtn = document.getElementById('showFormBtn');
            const cancelBtn = document.getElementById('cancelBtn'); const saveBtn = document.getElementById('saveBtn');
            const searchInput = document.getElementById('searchInput'); const licensesTableBody = document.querySelector('#licensesTable tbody');
            const buildingTypeSelect = document.getElementById('buildingType'); const floorGroup = document.getElementById('floor-group');
            const exportBtn = document.getElementById('exportData'); const importBtn = document.getElementById('importDataBtn');
            const importFile = document.getElementById('importFile'); const printAllBtn = document.getElementById('printAll');
            const form = document.getElementById('license-form').querySelector('form');
            const attachmentsInput = document.getElementById('attachments'); const attachmentsList = document.getElementById('attachments-list');

            let licenses = JSON.parse(localStorage.getItem('licensesData_v6')) || []; let currentlyEditingIndex = null; let fileAttachments = [];

            const showToast = (message) => {
                const toast = document.getElementById('toast-notification'); toast.textContent = message; toast.classList.add('show');
                setTimeout(() => { toast.classList.remove('show'); }, 3000);
            };

            const renderTable = () => {
                licensesTableBody.innerHTML = ''; const searchTerm = searchInput.value.toLowerCase();
                const filteredLicenses = licenses.filter(license => license.licenseId.toLowerCase().startsWith(searchTerm));
                if (filteredLicenses.length === 0) { licensesTableBody.innerHTML = `<tr><td colspan="5" style="text-align:center; padding: 20px;">لا توجد بيانات لعرضها. اضغط على 'إضافة رخصة جديدة' للبدء.</td></tr>`; return; }
                filteredLicenses.forEach(license => {
                    const originalIndex = licenses.findIndex(l => l.id === license.id);
                    const tr = document.createElement('tr');
                    tr.innerHTML = `<td>${escapeHTML(license.licenseId)}</td><td>${escapeHTML(license.applicantName)}</td><td>${escapeHTML(license.location)}</td><td>${escapeHTML(license.issueDate)}</td><td class="actions-cell"><button class="btn-warning" onclick="editLicense(${originalIndex})">تعديل</button><button class="btn-info" onclick="printLicense(${originalIndex})">طباعة</button><button class="btn-danger" onclick="deleteLicense(${originalIndex})">حذف</button></td>`;
                    licensesTableBody.appendChild(tr);
                });
            };
            
            const saveData = () => { localStorage.setItem('licensesData_v6', JSON.stringify(licenses)); };
            const clearForm = () => { form.reset(); currentlyEditingIndex = null; fileAttachments = []; attachmentsList.innerHTML = ''; saveBtn.textContent = '💾 حفظ'; saveBtn.classList.remove('btn-warning'); saveBtn.classList.add('btn-success'); buildingTypeSelect.dispatchEvent(new Event('change')); };
            const handleFileUpload = (files) => { return Promise.all(Array.from(files).map(file => new Promise((resolve, reject) => { const reader = new FileReader(); reader.onload = e => resolve({ name: file.name, type: file.type, data: e.target.result }); reader.onerror = e => reject(e); reader.readAsDataURL(file); }))); };
            const escapeHTML = str => str ? str.toString().replace(/[&<>"']/g, m => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' })[m]) : '';
            
            showFormBtn.addEventListener('click', () => { clearForm(); licenseForm.style.display = 'block'; showFormBtn.style.display = 'none'; saveBtn.textContent = '💾 حفظ'; });
            cancelBtn.addEventListener('click', () => { licenseForm.style.display = 'none'; showFormBtn.style.display = 'block'; });

            attachmentsInput.addEventListener('change', () => {
                attachmentsList.innerHTML = '';
                if(attachmentsInput.files.length > 0) { attachmentsList.textContent = `الملفات المختارة: ${Array.from(attachmentsInput.files).map(f => f.name).join(', ')}`; }
            });

            saveBtn.addEventListener('click', async () => {
                const licenseIdInput = document.getElementById('licenseId'); if (!licenseIdInput.value.trim()) { alert('يرجى إدخال رقم الرخصة.'); licenseIdInput.focus(); return; }
                const isDuplicate = licenses.some((license, index) => license.licenseId === licenseIdInput.value.trim() && index !== currentlyEditingIndex);
                if(isDuplicate) { alert('رقم الرخصة هذا موجود بالفعل. يرجى إدخال رقم فريد.'); licenseIdInput.focus(); return; }
                if (attachmentsInput.files.length > 0) { try { fileAttachments = await handleFileUpload(attachmentsInput.files); } catch (error) { alert("حدث خطأ أثناء تحميل الملفات."); return; } }
                const licenseData = { id: currentlyEditingIndex !== null ? licenses[currentlyEditingIndex].id : Date.now(), licenseId: document.getElementById('licenseId').value, licenseType: document.querySelector('input[name="licenseType"]:checked').value, buildingType: document.getElementById('buildingType').value, floorLevel: floorGroup.style.display !== 'none' ? document.getElementById('floorLevel').value : null, location: document.getElementById('location').value, applicantName: document.getElementById('applicantName').value, applicantId: document.getElementById('applicantId').value, applicantPhone: document.getElementById('applicantPhone').value, borders: { north: document.getElementById('borderNorth').value, south: document.getElementById('borderSouth').value, east: document.getElementById('borderEast').value, west: document.getElementById('borderWest').value }, dimensions: { north: document.getElementById('dimNorth').value, south: document.getElementById('dimSouth').value, east: document.getElementById('dimEast').value, west: document.getElementById('dimWest').value }, area: document.getElementById('area').value, issueDate: document.getElementById('issueDate').value, fees: { build: { voucher: document.getElementById('feeBuildVoucher').value, date: document.getElementById('feeBuildDate').value, amount: document.getElementById('feeBuildAmount').value }, plan: { voucher: document.getElementById('feePlanVoucher').value, date: document.getElementById('feePlanDate').value, amount: document.getElementById('feePlanAmount').value }, waste: { voucher: document.getElementById('feeWasteVoucher').value, date: document.getElementById('feeWasteDate').value, amount: document.getElementById('feeWasteAmount').value }, improve: { voucher: document.getElementById('feeImproveVoucher').value, date: document.getElementById('feeImproveDate').value, amount: document.getElementById('feeImproveAmount').value } }, attachments: fileAttachments };
                if (currentlyEditingIndex !== null) { if(fileAttachments.length === 0 && licenses[currentlyEditingIndex].attachments) { licenseData.attachments = licenses[currentlyEditingIndex].attachments; } licenses[currentlyEditingIndex] = licenseData; } else { licenses.push(licenseData); }
                saveData(); renderTable(); showToast(currentlyEditingIndex !== null ? 'تم تحديث الرخصة بنجاح!' : 'تم حفظ رخصة جديدة بنجاح!'); licenseForm.style.display = 'none'; showFormBtn.style.display = 'block';
            });

            searchInput.addEventListener('input', renderTable);
            buildingTypeSelect.addEventListener('change', () => { floorGroup.style.display = ['مبنى سكني', 'مبنى تجاري'].includes(buildingTypeSelect.value) ? 'block' : 'none'; });
            exportBtn.addEventListener('click', () => { if (licenses.length === 0) return alert('لا توجد بيانات لتصديرها.'); const dataStr = JSON.stringify(licenses, null, 2); const dataBlob = new Blob([dataStr], { type: 'application/json' }); const url = URL.createObjectURL(dataBlob); const a = document.createElement('a'); const timestamp = new Date().toISOString().slice(0, 19).replace(/:/g, '-'); a.href = url; a.download = `تصدير_بيانات_التراخيص_${timestamp}.json`; a.click(); URL.revokeObjectURL(url); });
            importBtn.addEventListener('click', () => importFile.click());
            importFile.addEventListener('change', (event) => {
                const file = event.target.files[0]; if (!file) return;
                const reader = new FileReader();
                reader.onload = (e) => {
                    try {
                        const importedData = JSON.parse(e.target.result); if (!Array.isArray(importedData)) throw new Error("Invalid format");
                        const choice = confirm("اختر طريقة الاستيراد:\n\n- اضغط 'موافق' (OK) لإضافة البيانات للبيانات الحالية.\n- اضغط 'إلغاء' (Cancel) لحذف البيانات الحالية واستبدالها بالجديدة.");
                        let importCount = 0;
                        if (choice) { const existingIds = new Set(licenses.map(l => l.id)); const newData = importedData.filter(imp => !existingIds.has(imp.id)); licenses = [...licenses, ...newData]; importCount = newData.length; } 
                        else { licenses = importedData; importCount = importedData.length; }
                        saveData(); renderTable(); alert(`اكتمل الاستيراد بنجاح!\nتمت معالجة ${importCount} سجل.`);
                    } catch (error) { alert('خطأ في استيراد الملف. تأكد من أن الملف بصيغة JSON صحيحة.'); } finally { importFile.value = ''; }
                };
                reader.readAsText(file);
            });
            
            // --- التعديل هنا ---
            printAllBtn.addEventListener('click', () => {
                if (licenses.length === 0) {
                    alert('لا توجد تراخيص لطباعتها.');
                    return;
                }

                let printStyles = `
                    @import url('https://fonts.googleapis.com/css2?family=Tajawal:wght@400;700&display=swap');
                    body { font-family: 'Tajawal', sans-serif; direction: rtl; margin: 20px; }
                    @page { size: A4; margin: 20mm; }
                    .print-header { background-color: #f8f9fa; border: 1px solid #dee2e6; border-radius: 12px; padding: 25px; text-align: center; margin-bottom: 30px; box-shadow: 0 4px 8px rgba(0,0,0,0.05); }
                    .print-header h1 { margin: 0 0 10px 0; font-size: 24px; color: #212529; }
                    .print-header h2 { margin: 0; font-size: 18px; color: #495057; font-weight: normal; }
                    .print-table { width: 100%; border-collapse: collapse; font-size: 12pt; }
                    .print-table th, .print-table td { border: 1px solid #adb5bd; padding: 10px; text-align: right; }
                    .print-table th { background-color: #e9ecef; font-weight: bold; }
                `;

                let tableRows = licenses.map(license => `
                    <tr>
                        <td>${escapeHTML(license.licenseId)}</td>
                        <td>${escapeHTML(license.applicantName)}</td>
                        <td>${escapeHTML(license.buildingType)}</td>
                        <td>${escapeHTML(license.location)}</td>
                        <td>${escapeHTML(license.issueDate)}</td>
                    </tr>
                `).join('');

                let printContent = `
                    <div class="print-header">
                        <h1>&#x1F3E2; مكتب الأشغال العامة والطرق</h1>
                        <h2>تقرير جميع التراخيص</h2>
                    </div>
                    <table class="print-table">
                        <thead>
                            <tr>
                                <th>رقم الرخصة</th>
                                <th>اسم طالب الرخصة</th>
                                <th>نوع المبنى</th>
                                <th>الموقع</th>
                                <th>تاريخ الإصدار</th>
                            </tr>
                        </thead>
                        <tbody>
                            ${tableRows}
                        </tbody>
                    </table>
                `;

                const printWindow = window.open('', '_blank');
                printWindow.document.write('<html><head><title>تقرير جميع التراخيص</title><style>' + printStyles + '</style></head><body>' + printContent + '</body></html>');
                printWindow.document.close();
                setTimeout(() => { printWindow.focus(); printWindow.print(); printWindow.close(); }, 500);
            });
            // --- نهاية التعديل ---

            window.editLicense = (index) => {
                clearForm(); licenseForm.style.display = 'block'; showFormBtn.style.display = 'none';
                currentlyEditingIndex = index; const license = licenses[index];
                document.getElementById('licenseId').value = license.licenseId; document.querySelector(`input[name="licenseType"][value="${license.licenseType}"]`).checked = true;
                document.getElementById('buildingType').value = license.buildingType; buildingTypeSelect.dispatchEvent(new Event('change'));
                if(license.floorLevel) document.getElementById('floorLevel').value = license.floorLevel;
                document.getElementById('location').value = license.location; document.getElementById('applicantName').value = license.applicantName;
                document.getElementById('applicantId').value = license.applicantId; document.getElementById('applicantPhone').value = license.applicantPhone;
                Object.keys(license.borders).forEach(key => document.getElementById('border' + key.charAt(0).toUpperCase() + key.slice(1)).value = license.borders[key]);
                Object.keys(license.dimensions).forEach(key => document.getElementById('dim' + key.charAt(0).toUpperCase() + key.slice(1)).value = license.dimensions[key]);
                document.getElementById('area').value = license.area; document.getElementById('issueDate').value = license.issueDate;
                Object.keys(license.fees).forEach(type => Object.keys(license.fees[type]).forEach(prop => { const el = document.getElementById('fee' + type.charAt(0).toUpperCase() + type.slice(1) + prop.charAt(0).toUpperCase() + prop.slice(1)); if(el) el.value = license.fees[type][prop]; }));
                fileAttachments = license.attachments || [];
                if (fileAttachments.length > 0) { attachmentsList.textContent = `الملفات المحفوظة: ${fileAttachments.map(f => f.name).join(', ')}`; }
                saveBtn.textContent = '💾 تحديث'; saveBtn.classList.replace('btn-success', 'btn-warning'); window.scrollTo({ top: 0, behavior: 'smooth' });
            };

            window.deleteLicense = (index) => { if (confirm(`هل أنت متأكد من حذف الرخصة رقم ${licenses[index].licenseId}؟`)) { licenses.splice(index, 1); saveData(); renderTable(); } };
            
            window.printLicense = (index) => {
                const license = licenses[index];
                
                let printStyles = `
                    @import url('https://fonts.googleapis.com/css2?family=Tajawal:wght@400;700&display=swap');
                    body { font-family: 'Tajawal', sans-serif; direction: rtl; margin: 0; padding: 0; }
                    @page { size: A4; margin: 20mm; }
                    .print-page { page-break-after: always; }
                    .print-page:last-child { page-break-after: avoid; }
                    h1, h2 { text-align: center; color: #333; }
                    .print-table { width: 100%; border-collapse: collapse; font-size: 12pt; margin-top: 20px; }
                    .print-table td { border: 1px solid #333; padding: 8px; text-align: right; }
                    .print-table td:first-child { font-weight: bold; background-color: #f0f0f0; width: 150px; }
                    .attachment-page { page-break-before: always; width: 100%; height: 100%; display: flex; justify-content: center; align-items: center; padding: 0; margin: 0; }
                    .attachment-page img, .attachment-page iframe { max-width: 100%; max-height: 100%; width: 100%; height: 100vh; border: none; object-fit: contain; }
                `;

                let licenseDetailsHtml = `<div class="print-page"><h1>مكتب الأشغال العامة والطرق - قسم تراخيص البناء</h1><h2>تفاصيل الرخصة رقم: ${escapeHTML(license.licenseId)}</h2><table class="print-table"><tr><td>رقم الرخصة</td><td>${escapeHTML(license.licenseId)}</td></tr><tr><td>نوع الرخصة</td><td>${escapeHTML(license.licenseType)}</td></tr><tr><td>نوع المبنى</td><td>${escapeHTML(license.buildingType)} ${license.floorLevel ? `(${escapeHTML(license.floorLevel)})` : ''}</td></tr><tr><td>تاريخ الإصدار</td><td>${escapeHTML(license.issueDate)}</td></tr><tr><td>اسم طالب الرخصة</td><td>${escapeHTML(license.applicantName)}</td></tr><tr><td>رقم البطاقة/الجواز</td><td>${escapeHTML(license.applicantId)}</td></tr><tr><td>رقم الهاتف</td><td>${escapeHTML(license.applicantPhone)}</td></tr><tr><td>موقع المبنى</td><td>${escapeHTML(license.location)}</td></tr><tr><td>المساحة</td><td>${escapeHTML(license.area)} م²</td></tr><tr><td>الحدود</td><td>ش: ${escapeHTML(license.borders.north)}, ج: ${escapeHTML(license.borders.south)}, ش: ${escapeHTML(license.borders.east)}, غ: ${escapeHTML(license.borders.west)}</td></tr><tr><td>الأبعاد</td><td>ش: ${escapeHTML(license.dimensions.north)}م, ج: ${escapeHTML(license.dimensions.south)}م, ش: ${escapeHTML(license.dimensions.east)}م, غ: ${escapeHTML(license.dimensions.west)}م</td></tr></table><h2>تفاصيل الرسوم المالية</h2><table class="print-table"><thead><tr><th>البيان</th><th>رقم السند</th><th>تاريخ السند</th><th>المبلغ</th></tr></thead><tbody><tr><td>رسوم البناء</td><td>${escapeHTML(license.fees.build.voucher)}</td><td>${escapeHTML(license.fees.build.date)}</td><td>${escapeHTML(license.fees.build.amount)}</td></tr><tr><td>رسوم التخطيط</td><td>${escapeHTML(license.fees.plan.voucher)}</td><td>${escapeHTML(license.fees.plan.date)}</td><td>${escapeHTML(license.fees.plan.amount)}</td></tr><tr><td>رسوم رفع المخلفات</td><td>${escapeHTML(license.fees.waste.voucher)}</td><td>${escapeHTML(license.fees.waste.date)}</td><td>${escapeHTML(license.fees.waste.amount)}</td></tr><tr><td>رسوم التحسين</td><td>${escapeHTML(license.fees.improve.voucher)}</td><td>${escapeHTML(license.fees.improve.date)}</td><td>${escapeHTML(license.fees.improve.amount)}</td></tr></tbody></table></div>`;
                
                let attachmentsHtml = '';
                if (license.attachments && license.attachments.length > 0) {
                    license.attachments.forEach(file => { attachmentsHtml += `<div class="attachment-page">${file.type.startsWith('image/') ? `<img src="${file.data}">` : `<iframe src="${file.data}"></iframe>`}</div>`; });
                }

                const printWindow = window.open('', '_blank');
                printWindow.document.write(`<html><head><title>طباعة رخصة: ${escapeHTML(license.licenseId)}</title><style>${printStyles}</style></head><body>${licenseDetailsHtml}${attachmentsHtml}</body></html>`);
                printWindow.document.close();
                setTimeout(() => { printWindow.focus(); printWindow.print(); printWindow.close(); }, 500);
            };

            renderTable(); buildingTypeSelect.dispatchEvent(new Event('change'));
        });
    </script>
</body>
</html>





index.html
<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>تسجيل الدخول - مكتب الأشغال العامة والطرق</title>
    <style>
        /* CSS مدمج مباشرة هنا لضمان عمله */
        @import url('https://fonts.googleapis.com/css2?family=Tajawal:wght@400;700&display=swap');
        :root {
            --primary-color: #4a69bd; --secondary-color: #6a89cc; --accent-color: #f6b93b;
            --background-start: #f7f7f7; --background-end: #e2e8f0; --surface-color: rgba(255, 255, 255, 0.9);
            --text-color: #1e272e; --danger-color: #e55039; --success-color: #38ada9;
            --border-radius: 12px; --shadow: 0 10px 30px rgba(0, 0, 0, 0.1);
        }
        * { box-sizing: border-box; margin: 0; padding: 0; }
        body {
            font-family: 'Tajawal', sans-serif; direction: rtl;
            background-image: linear-gradient(to top left, var(--background-start), var(--background-end));
            color: var(--text-color); line-height: 1.7; padding: 20px; min-height: 100vh;
        }
        .container { max-width: 1200px; margin: auto; background: var(--surface-color); padding: 30px; border-radius: var(--border-radius); box-shadow: var(--shadow); border: 1px solid rgba(255, 255, 255, 0.2); }
        header { text-align: center; padding: 20px; background-image: linear-gradient(45deg, var(--primary-color), var(--secondary-color)); color: white; border-radius: var(--border-radius); margin-bottom: 25px; text-shadow: 1px 1px 3px rgba(0,0,0,0.2); }
        header h1 { margin: 0; font-size: 2.2rem; }
        header p { margin: 5px 0 0; font-size: 1.3rem; opacity: 0.9; }
        footer { text-align: center; margin-top: 30px; padding: 15px; color: var(--text-color); opacity: 0.7; }
        footer a { color: var(--primary-color); font-weight: bold; text-decoration: none;}
        .login-container { max-width: 450px; margin: 5vh auto; padding: 40px; text-align: center; }
        .login-container .form-group { text-align: right; margin-bottom: 15px; }
        input, select, textarea { width: 100%; padding: 14px; border: 1px solid #ddd; border-radius: var(--border-radius); font-family: 'Tajawal', sans-serif; font-size: 1rem; transition: all 0.3s ease; background-color: #fcfcfc; }
        input:focus, select:focus, textarea:focus { outline: none; border-color: var(--primary-color); box-shadow: 0 0 0 3px rgba(74, 105, 189, 0.2); }
        button, .btn { padding: 14px 22px; border: none; border-radius: var(--border-radius); background-color: var(--primary-color); color: white; font-family: inherit; font-size: 1rem; font-weight: 700; cursor: pointer; transition: all 0.3s ease; box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1); display: inline-flex; align-items: center; justify-content: center; gap: 8px; }
        button:hover, .btn:hover { transform: translateY(-3px) scale(1.03); box-shadow: 0 8px 20px rgba(0, 0, 0, 0.15); }
        button:active { transform: translateY(-1px) scale(1); }
        button.btn-success { background-color: var(--success-color); }
        .form-group label { margin-bottom: 8px; font-weight: 700; font-size: 0.95rem; display: block; }
        .modal { display: none; position: fixed; z-index: 1000; left: 0; top: 0; width: 100%; height: 100%; background-color: rgba(0,0,0,0.6); animation: fadeIn 0.3s ease; }
        .modal-content { background-color: #fff; margin: 15% auto; padding: 30px; border-radius: var(--border-radius); width: 90%; max-width: 500px; box-shadow: var(--shadow); position: relative; animation: slideIn 0.4s ease; }
        @keyframes fadeIn { from { opacity: 0; } to { opacity: 1; } }
        @keyframes slideIn { from { transform: translateY(-50px); opacity: 0; } to { transform: translateY(0); opacity: 1; } }
        .close-btn { color: #aaa; position: absolute; left: 20px; top: 10px; font-size: 28px; font-weight: bold; cursor: pointer; transition: color 0.3s ease; }
        .close-btn:hover { color: var(--danger-color); }
    </style>
</head>
<body>
    <div class="container login-container">
        <header>
            <h1>مكتب الأشغال العامة والطرق<br><small>قسم تراخيص البناء</small></h1>
        </header>
        <div class="form-group">
            <label for="username">اسم المستخدم</label>
            <input type="text" id="username" value="admin">
        </div>
        <div class="form-group">
            <label for="password">كلمة المرور</label>
            <input type="password" id="password" placeholder="ادخل كلمة المرور">
        </div>
        <button id="loginBtn" class="btn-success" style="width: 100%; margin-top: 10px;">تسجيل الدخول</button>
        <button id="changePasswordBtn" style="width: 100%; margin-top: 10px;">تعديل كلمة المرور</button>
    </div>
    <div id="passwordModal" class="modal">
        <div class="modal-content">
            <span class="close-btn">&times;</span>
            <h2>تعديل كلمة المرور</h2>
            <div class="form-group"><label for="currentPassword">كلمة المرور الحالية</label><input type="password" id="currentPassword" required></div>
            <div class="form-group"><label for="newPassword">كلمة المرور الجديدة</label><input type="password" id="newPassword" required></div>
            <div class="form-group"><label for="confirmPassword">تأكيد كلمة المرور الجديدة</label><input type="password" id="confirmPassword" required></div>
            <button id="savePasswordBtn" class="btn-success">حفظ التغييرات</button>
        </div>
    </div>
    <footer>
        <p>إعداد: م. وليد غنيمة | <a href="tel:+967775044084">+967775044084</a> | <a href="mailto:wajag93@gmail.com">wajag93@gmail.com</a></p>
    </footer>

    <script>
        // JS مدمج مباشرة هنا لضمان عمله
        document.addEventListener('DOMContentLoaded', () => {
            const loginBtn = document.getElementById('loginBtn');
            const changePasswordBtn = document.getElementById('changePasswordBtn');
            const modal = document.getElementById('passwordModal');
            const closeModal = document.querySelector('.close-btn');
            const savePasswordBtn = document.getElementById('savePasswordBtn');
            if (!localStorage.getItem('buildingPermitPassword')) { localStorage.setItem('buildingPermitPassword', 'wag123'); }
            loginBtn.addEventListener('click', () => {
                const username = document.getElementById('username').value;
                const password = document.getElementById('password').value;
                const storedPassword = localStorage.getItem('buildingPermitPassword');
                if (username === 'admin' && password === storedPassword) { window.location.href = 'dashboard.html'; } 
                else { alert('اسم المستخدم أو كلمة المرور غير صحيحة!'); }
            });
            document.getElementById('password').addEventListener('keypress', function(e) { if (e.key === 'Enter') { loginBtn.click(); } });
            changePasswordBtn.addEventListener('click', () => { modal.style.display = 'block'; });
            closeModal.addEventListener('click', () => { modal.style.display = 'none'; });
            savePasswordBtn.addEventListener('click', () => {
                const currentPassword = document.getElementById('currentPassword').value;
                const newPassword = document.getElementById('newPassword').value;
                const confirmPassword = document.getElementById('confirmPassword').value;
                const storedPassword = localStorage.getItem('buildingPermitPassword');
                if(currentPassword !== storedPassword) { alert('كلمة المرور الحالية غير صحيحة!'); return; }
                if(!newPassword || newPassword.length < 4){ alert('كلمة المرور الجديدة يجب أن تكون 4 أحرف على الأقل.'); return; }
                if (newPassword !== confirmPassword) { alert('كلمتا المرور الجديدتان غير متطابقتين!'); return; }
                localStorage.setItem('buildingPermitPassword', newPassword);
                alert('تم تغيير كلمة المرور بنجاح!');
                modal.style.display = 'none';
                document.getElementById('currentPassword').value = ''; document.getElementById('newPassword').value = ''; document.getElementById('confirmPassword').value = '';
            });
            window.onclick = function(event) { if (event.target == modal) { modal.style.display = "none"; } }
        });
    </script>
</body>
</html>




styles.css
/* --- تصميم جديد ومحسّن --- */
@import url('https://fonts.googleapis.com/css2?family=Tajawal:wght@400;700&display=swap');

:root {
    --primary-color: #4a69bd; /* أزرق هادئ */
    --secondary-color: #6a89cc;
    --accent-color: #f6b93b; /* أصفر ذهبي */
    --background-start: #f7f7f7;
    --background-end: #e2e8f0;
    --surface-color: rgba(255, 255, 255, 0.9); /* سطح شبه شفاف */
    --text-color: #1e272e;
    --danger-color: #e55039;
    --success-color: #38ada9;
    --border-radius: 12px;
    --shadow: 0 10px 30px rgba(0, 0, 0, 0.1);
}

* { box-sizing: border-box; margin: 0; padding: 0; }

body {
    font-family: 'Tajawal', sans-serif;
    direction: rtl;
    background-image: linear-gradient(to top left, var(--background-start), var(--background-end));
    color: var(--text-color);
    line-height: 1.7;
    padding: 20px;
    min-height: 100vh;
}

.container {
    max-width: 1200px;
    margin: auto;
    background: var(--surface-color);
    padding: 30px;
    border-radius: var(--border-radius);
    box-shadow: var(--shadow);
    backdrop-filter: blur(10px); /* تأثير زجاجي */
    border: 1px solid rgba(255, 255, 255, 0.2);
}

header {
    text-align: center;
    padding: 20px;
    background-image: linear-gradient(45deg, var(--primary-color), var(--secondary-color));
    color: white;
    border-radius: var(--border-radius);
    margin-bottom: 25px;
    text-shadow: 1px 1px 3px rgba(0,0,0,0.2);
}

header h1 { margin: 0; font-size: 2.2rem; }
header p { margin: 5px 0 0; font-size: 1.3rem; opacity: 0.9; }

footer { text-align: center; margin-top: 30px; padding: 15px; color: var(--text-color); opacity: 0.7; }
footer a { color: var(--primary-color); font-weight: bold; text-decoration: none;}

.login-container { max-width: 450px; margin: 5vh auto; padding: 40px; text-align: center; }
.login-container header { margin: -40px -40px 30px -40px; border-radius: 12px 12px 0 0; }
.login-container .form-group { text-align: right; margin-bottom: 15px; }

.controls, .main-actions { display: flex; flex-wrap: wrap; gap: 15px; margin-bottom: 25px; align-items: center; }

input, select, textarea {
    width: 100%;
    padding: 14px;
    border: 1px solid #ddd;
    border-radius: var(--border-radius);
    font-family: 'Tajawal', sans-serif;
    font-size: 1rem;
    transition: all 0.3s ease;
    background-color: #fcfcfc;
}
input:focus, select:focus, textarea:focus { outline: none; border-color: var(--primary-color); box-shadow: 0 0 0 3px rgba(74, 105, 189, 0.2); }

button, .btn {
    padding: 14px 22px;
    border: none;
    border-radius: var(--border-radius);
    background-color: var(--primary-color);
    color: white;
    font-family: inherit;
    font-size: 1rem;
    font-weight: 700;
    cursor: pointer;
    transition: all 0.3s ease;
    box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
    display: inline-flex;
    align-items: center;
    justify-content: center;
    gap: 8px;
}
button:hover, .btn:hover { transform: translateY(-3px) scale(1.03); box-shadow: 0 8px 20px rgba(0, 0, 0, 0.15); }
button:active { transform: translateY(-1px) scale(1); }

button.btn-success { background-color: var(--success-color); }
button.btn-danger { background-color: var(--danger-color); }
button.btn-warning { background-color: var(--accent-color); color: #333; }
button.btn-info { background-color: var(--secondary-color); }

#license-form { background: transparent; padding: 0; border: none; margin-bottom: 30px; }
#license-form h2 { margin-bottom: 20px; padding-bottom: 10px; border-bottom: 2px solid var(--primary-color); color: var(--primary-color); font-size: 1.8rem; }

.form-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(280px, 1fr)); gap: 25px; }
.form-group label { margin-bottom: 8px; font-weight: 700; font-size: 0.95rem; display: block; }
.radio-group { display: flex; gap: 20px; align-items: center; padding-top: 10px; }

fieldset { border: 1px solid #ddd; border-radius: var(--border-radius); padding: 20px; margin-top: 10px; transition: all 0.3s ease; }
fieldset:hover { border-color: var(--secondary-color); }
fieldset legend { padding: 0 10px; font-weight: 700; color: var(--primary-color); }

.table-container { overflow-x: auto; border: 1px solid #e0e0e0; border-radius: var(--border-radius); box-shadow: 0 4px 10px rgba(0,0,0,0.05); }
table { width: 100%; border-collapse: collapse; }
th, td { padding: 16px; text-align: right; border-bottom: 1px solid #e0e0e0; }
thead th { background-color: #f8f9fa; color: var(--text-color); font-size: 1.1rem; position: sticky; top: 0; }
tbody tr { transition: background-color 0.3s ease; }
tbody tr:last-child td { border-bottom: none; }
tbody tr:hover { background-color: rgba(106, 137, 204, 0.1); }
.actions-cell button { padding: 8px 12px; margin: 0 4px; font-size: 0.9rem; }

.modal { display: none; position: fixed; z-index: 1000; left: 0; top: 0; width: 100%; height: 100%; background-color: rgba(0,0,0,0.6); animation: fadeIn 0.3s ease; }
.modal-content { background-color: #fff; margin: 15% auto; padding: 30px; border-radius: var(--border-radius); width: 90%; max-width: 500px; box-shadow: var(--shadow); position: relative; animation: slideIn 0.4s ease; }
@keyframes fadeIn { from { opacity: 0; } to { opacity: 1; } }
@keyframes slideIn { from { transform: translateY(-50px); opacity: 0; } to { transform: translateY(0); opacity: 1; } }
.close-btn { color: #aaa; position: absolute; left: 20px; top: 10px; font-size: 28px; font-weight: bold; cursor: pointer; transition: color 0.3s ease; }
.close-btn:hover { color: var(--danger-color); }

@media print {
    body { padding: 0; background: #fff; }
    .no-print, header, footer, #license-form, .main-actions, .controls, .actions-cell { display: none !important; }
    .container { box-shadow: none; padding: 0; border: none; }
    table, th, td { border: 1px solid #ccc !important; }
    #print-area { display: block !important; }
    #print-area h1, #print-area h2 { text-align: center; color: black; background: none; }
    .print-table { width: 100%; border-collapse: collapse; font-size: 12pt; margin-top: 20px; }
    .print-table td { border: 1px solid #333; padding: 8px; }
    .print-table td:first-child { font-weight: bold; background-color: #f0f0f0; width: 200px; }
    .print-attachments img, .print-attachments iframe { max-width: 100%; display: block; margin-top: 15px; page-break-before: always; }
    .print-attachments h2 { page-break-before: always; padding-top: 20px; }
}

