import pandas as pd
import math
import streamlit as st

class Analyzer:
    def __init__(self, code, name, tests_per_hour, cost, amortization, work_hours=8, work_days=22):
        self.code = code
        self.name = name
        self.tests_per_hour = tests_per_hour
        self.cost = cost
        self.amortization = amortization
        self.work_hours = work_hours
        self.work_days = work_days
        self.studies = []

class Study:
    def __init__(self, code, name, research_flow, error_rate, control_tests):
        self.code = code
        self.name = name
        self.research_flow = research_flow
        self.error_rate = error_rate
        self.control_tests = control_tests

class MedicalCostCalculator:
    def __init__(self):
        self.analyzers = []
        self.common_materials = pd.DataFrame(columns=[
            'Анализатор', 'Наименование', 'Код', 'Ед. изм.', 'Фасовка',
            'Расход на тест', 'Цена за упаковку', 'Стоимость'
        ])
        self.unique_materials = pd.DataFrame(columns=[
            'Анализатор', 'Исследование', 'Наименование', 'Код',
            'Ед. изм.', 'Фасовка', 'Расход на тест', 'Цена за упаковку', 'Стоимость'
        ])
        self.labor = pd.DataFrame(columns=[
            'Должность', 'Время (мин/тест)', 'Зарплата',
            'Нормочасов', 'Стоимость', 'Сотрудников', 'Часы/день', 'Загрузка (%)'
        ])

    def add_analyzer(self, analyzer):
        self.analyzers.append(analyzer)

    def add_study_to_analyzer(self, analyzer_code, study):
        for analyzer in self.analyzers:
            if analyzer.code == analyzer_code:
                analyzer.studies.append(study)
                return
        raise ValueError(f"Анализатор с кодом {analyzer_code} не найден")

    def _calculate_total_tests(self):
        total = 0
        for analyzer in self.analyzers:
            for study in analyzer.studies:
                total += study.research_flow * (1 + study.error_rate) + study.control_tests
        return total

    def add_common_material(self, analyzer_code, name, code, unit, package_size, usage_per_test, package_price):
        analyzer = next((a for a in self.analyzers if a.code == analyzer_code), None)
        if not analyzer:
            raise ValueError(f"Анализатор с кодом {analyzer_code} не найден")
        
        total_tests_analyzer = sum(
            (study.research_flow * (1 + study.error_rate) + study.control_tests
            for study in analyzer.studies
        )
        
        total_usage = usage_per_test * total_tests_analyzer
        packages_needed = math.ceil(total_usage / package_size)
        
        total_research_flow_analyzer = sum(study.research_flow for study in analyzer.studies)
        if total_research_flow_analyzer == 0:
            raise ValueError(f"Нет исследований для анализатора {analyzer_code}. Добавьте исследования перед материалами.")
        
        cost_per_service = (packages_needed * package_price) / total_research_flow_analyzer
        
        self.common_materials.loc[len(self.common_materials)] = [
            analyzer.name, name, code, unit, package_size,
            usage_per_test, package_price, round(cost_per_service, 2)
        ]

    def add_unique_material(self, analyzer_code, study_code, name, code, unit, package_size, usage_per_test, package_price):
        for analyzer in self.analyzers:
            if analyzer.code == analyzer_code:
                for study in analyzer.studies:
                    if study.code == study_code:
                        total_usage = usage_per_test * (
                            study.research_flow * (1 + study.error_rate) + study.control_tests
                        )
                        packages_needed = math.ceil(total_usage / package_size)
                        cost_per_service = (packages_needed * package_price) / study.research_flow
                        self.unique_materials.loc[len(self.unique_materials)] = [
                            analyzer.name, study.name, name, code,
                            unit, package_size, usage_per_test,
                            package_price, round(cost_per_service, 2)
                        ]
                        return
        raise ValueError("Анализатор или исследование не найдены")

    def add_labor(self, position, time_per_test, salary, hours_per_month, employees=1.0, hours_per_day=8):
        total_time_hours = (time_per_test * self._calculate_total_tests()) / 60
        available_hours = employees * hours_per_day * 22
        load_percent = (total_time_hours / available_hours) * 100 if available_hours > 0 else 0
        total_research_flow = sum(
            study.research_flow
            for analyzer in self.analyzers
            for study in analyzer.studies
        )
        if total_research_flow == 0:
            raise ValueError("Не добавлено ни одного исследования. Добавьте исследование перед добавлением персонала.")

        cost = (total_time_hours * (salary / hours_per_month)) / total_research_flow
        self.labor.loc[len(self.labor)] = [
            position, time_per_test, salary,
            hours_per_month, round(cost, 2),
            employees, hours_per_day,
            round(load_percent, 1)
        ]

    def load_from_excel(self, file):
        xls = pd.ExcelFile(file)
        
        # Загрузка анализаторов
        analyzers_df = pd.read_excel(xls, sheet_name="Анализаторы")
        for _, row in analyzers_df.iterrows():
            analyzer = Analyzer(
                code=str(row['Код*']).strip(),
                name=row['Название*'],
                tests_per_hour=float(str(row['Тестов/час*']).replace(',', '.')),
                cost=float(str(row['Стоимость (руб)*']).replace(',', '.')),
                amortization=float(str(row['Амортизация (мес)*']).replace(',', '.')),
                work_hours=float(str(row.get('Рабочие часы/день', 8)).replace(',', '.')),
                work_days=float(str(row.get('Рабочие дни/мес', 22)).replace(',', '.'))
            )
            self.add_analyzer(analyzer)

        # Загрузка исследований
        studies_df = pd.read_excel(xls, sheet_name="Исследования")
        for _, row in studies_df.iterrows():
            study = Study(
                code=str(row['Код исследования*']).strip(),
                name=row['Название*'],
                research_flow=float(str(row['Поток тестов/мес*']).replace(',', '.')),
                error_rate=float(str(row.get('Процент ошибок', 0)).replace(',', '.')),
                control_tests=int(row.get('Контрольные тесты', 0))
            )
            self.add_study_to_analyzer(str(row['Код анализатора*']).strip(), study)

        # Загрузка материалов
        common_materials_df = pd.read_excel(xls, sheet_name="Общие материалы")
        for _, row in common_materials_df.iterrows():
            self.add_common_material(
                analyzer_code=str(row['Код анализатора*']).strip(),
                name=row['Наименование*'],
                code=str(row['Код материала']).strip(),
                unit=row['Ед. измерения'],
                package_size=float(str(row['Фасовка']).replace(',', '.')),
                usage_per_test=float(str(row['Расход на тест']).replace(',', '.')),
                package_price=float(str(row['Цена за упаковку']).replace(',', '.'))
            )

        unique_materials_df = pd.read_excel(xls, sheet_name="Уникальные материалы")
        for _, row in unique_materials_df.iterrows():
            self.add_unique_material(
                analyzer_code=str(row['Код анализатора*']).strip(),
                study_code=str(row['Код исследования*']).strip(),
                name=row['Наименование*'],
                code=str(row['Код материала']).strip(),
                unit=row['Ед. измерения'],
                package_size=float(str(row['Фасовка']).replace(',', '.')),
                usage_per_test=float(str(row['Расход на тест']).replace(',', '.')),
                package_price=float(str(row['Цена за упаковку']).replace(',', '.'))
            )

        # Загрузка персонала
        labor_df = pd.read_excel(xls, sheet_name="Персонал")
        for _, row in labor_df.iterrows():
            self.add_labor(
                position=row['Должность*'],
                time_per_test=float(str(row['Время на тест (мин)*']).replace(',', '.')),
                salary=float(str(row['Зарплата (руб)*']).replace(',', '.')),
                hours_per_month=float(str(row.get('Нормочасов/мес', 160)).replace(',', '.')),
                employees=float(str(row.get('Сотрудников', 1)).replace(',', '.')),
                hours_per_day=float(str(row.get('Часы/день', 8)).replace(',', '.'))
            )

    def calculate_study_costs(self):
        study_costs = []
        total_research_flow = sum(
            study.research_flow 
            for analyzer in self.analyzers 
            for study in analyzer.studies
        )
        
        for analyzer in self.analyzers:
            analyzer_total_flow = sum(study.research_flow for study in analyzer.studies)
            
            for study in analyzer.studies:
                common_cost = sum(
                    row['Стоимость'] * (study.research_flow / analyzer_total_flow)
                    for _, row in self.common_materials.iterrows()
                    if row['Анализатор'] == analyzer.name
                )
                
                unique_cost = sum(
                    row['Стоимость'] 
                    for _, row in self.unique_materials.iterrows()
                    if row['Исследование'] == study.name
                )
                
                labor_cost = sum(
                    row['Стоимость'] * (study.research_flow / total_research_flow)
                    for _, row in self.labor.iterrows()
                )
                
                total_study_tests = study.research_flow * (1 + study.error_rate) + study.control_tests
                monthly_usage = (total_study_tests / analyzer.tests_per_hour) / (analyzer.work_hours * analyzer.work_days)
                equipment_cost = (analyzer.cost / analyzer.amortization) * monthly_usage / study.research_flow
                
                total_cost = common_cost + unique_cost + labor_cost + equipment_cost
                
                study_costs.append({
                    'Исследование': study.name,
                    'Код': study.code,
                    'Общие материалы': round(common_cost, 2),
                    'Уникальные материалы': round(unique_cost, 2),
                    'Трудозатраты': round(labor_cost, 2),
                    'Оборудование': round(equipment_cost, 2),
                    'Итого': round(total_cost, 2)
                })
        
        return study_costs

    def calculate_fot(self):
        fot_data = []
        total_monthly_fot = 0
        for _, row in self.labor.iterrows():
            monthly_salary = row['Зарплата'] * row['Сотрудников']
            total_monthly_fot += monthly_salary
            fot_data.append({
                'Должность': row['Должность'],
                'Кол-во сотрудников': row['Сотрудников'],
                'Зарплата на человека': row['Зарплата'],
                'Месячный ФОТ': monthly_salary,
                'Загрузка (%)': row['Загрузка (%)']
            })
        return pd.DataFrame(fot_data), total_monthly_fot

# Streamlit интерфейс
st.set_page_config(page_title="Медкалькулятор", layout="wide")
st.title("📊 Калькулятор себестоимости медицинских исследований")

uploaded_file = st.file_uploader("Загрузите файл Excel с данными", type=["xlsx"])

if uploaded_file:
    try:
        calculator = MedicalCostCalculator()
        calculator.load_from_excel(uploaded_file)
        
        st.success("Файл успешно загружен и обработан!")
        
        with st.expander("📋 Общая информация об анализаторах", expanded=True):
            for analyzer in calculator.analyzers:
                total_tests = sum(
                    study.research_flow * (1 + study.error_rate) + study.control_tests
                    for study in analyzer.studies
                )
                max_capacity = analyzer.tests_per_hour * analyzer.work_hours * analyzer.work_days
                load_percent = (total_tests / max_capacity) * 100 if max_capacity != 0 else 0
                
                cols = st.columns(3)
                cols[0].metric("Анализатор", analyzer.name)
                cols[1].metric("Загрузка", f"{load_percent:.1f}%")
                cols[2].metric("Всего тестов", int(total_tests))
                
                studies_df = pd.DataFrame([{
                    "Код": study.code,
                    "Исследование": study.name,
                    "Поток тестов": study.research_flow,
                    "Ошибки (%)": study.error_rate * 100,
                    "Контрольные тесты": study.control_tests
                } for study in analyzer.studies])
                
                st.dataframe(
                    studies_df,
                    column_config={
                        "Поток тестов": st.column_config.NumberColumn(format="%d"),
                        "Ошибки (%)": st.column_config.NumberColumn(format="%.1f %%"),
                        "Контрольные тесты": st.column_config.NumberColumn(format="%d")
                    },
                    hide_index=True
                )

        with st.expander("🧪 Материалы"):
            tab1, tab2 = st.tabs(["Общие материалы", "Уникальные материалы"])
            
            with tab1:
                st.dataframe(
                    calculator.common_materials,
                    column_config={"Стоимость": st.column_config.NumberColumn(format="%.2f ₽")},
                    hide_index=True
                )
            
            with tab2:
                st.dataframe(
                    calculator.unique_materials,
                    column_config={"Стоимость": st.column_config.NumberColumn(format="%.2f ₽")},
                    hide_index=True
                )

        with st.expander("👥 Персонал"):
            fot_df, total_fot = calculator.calculate_fot()
            cols = st.columns([3,1])
            
            cols[0].dataframe(
                fot_df,
                column_config={
                    "Месячный ФОТ": st.column_config.NumberColumn(format="%.2f ₽"),
                    "Загрузка (%)": st.column_config.ProgressColumn(format="%.1f")
                },
                hide_index=True
            )
            
            cols[1].metric("Общий ФОТ в месяц", f"{total_fot:,.2f} ₽")

        with st.expander("💰 Себестоимость исследований", expanded=True):
            study_costs = calculator.calculate_study_costs()
            df = pd.DataFrame(study_costs)
            
            st.dataframe(
                df.style.format({
                    'Общие материалы': '{:.2f} ₽',
                    'Уникальные материалы': '{:.2f} ₽',
                    'Трудозатраты': '{:.2f} ₽',
                    'Оборудование': '{:.2f} ₽',
                    'Итого': '{:.2f} ₽'
                }),
                column_config={
                    "Исследование": "Исследование",
                    "Код": "Код исследования"
                },
                use_container_width=True,
                hide_index=True
            )

    except Exception as e:
        st.error(f"Ошибка обработки файла: {str(e)}")
        st.stop()

st.caption("© Медкалькулятор 2024 | Для загрузки используйте шаблон Excel с соответствующими листами")


