ods graphics on;
libname librisk "/folders/myfolders/";
proc import datafile="/folders/myfolders/Credit_Data.csv" out=librisk.loan_data dbms=csv replace;
	getnames=yes;
	guessingrows=20;
run;

data risk_data;
	set librisk.loan_data;
	length income_category $10;
	if loan_status in ("Fully Paid", "Current", "In Grace Period") then Delinquency = 0;
	else Delinquency = 1;
	fico_avg = (fico_range_low+fico_range_high)/2;
	last_fico_avg = (last_fico_range_low+last_fico_range_high)/2;
	if annual_inc <= 47000 then income_category = 'Low';
	else if annual_inc > 47000 and annual_inc <= 95000 then income_category = 'Medium';
	else if annual_inc > 95000 and annual_inc <= 284000 then income_category = 'High';
	else if annual_inc > 284000 then income_category = 'Very High';
run;

proc contents data=risk_data; run;
proc means data=risk_data maxdec=2 Nmiss N; run;

proc freq data=risk_data(keep=term purpose emp_length home_ownership verification_status addr_state) nlevels; 
	title 'Unique values for character variable';
	tables _character_ / nocum nopercent;
run;

proc sgplot data=work.risk_data;
	title 'Does a higher loan amount result in loan default?';
	vbox Loan_amnt / category=Delinquency;
	refline 15711.51;
run;

proc sgplot data=risk_data;
	title 'Which type of loan has a higher risk of default?';
	vbar purpose / group=Delinquency groupdisplay=cluster;
run;

data income_no_out;
	set risk_data(keep=annual_inc installment loan_amnt income_category Delinquency);
	if annual_inc>284000 then delete;
	rename annual_inc = Annual_Income loan_amnt= Loan_Amount installment=EMI;
run;

proc sgplot data=income_no_out;
	title 'Distribution of Annual Income';
	histogram Annual_Income;
run;

data risk_data; set risk_data; if purpose = 'Other' then delete; run;

proc sgpie data = risk_data; title 'Type of loan'; donut purpose / datalabelloc=outside datalabeldisplay=(percent); run;

proc surveyselect data=income_no_out method=srs rep=1 sampsize=10000 seed=1 out=loan; run;

proc sgscatter data=loan; title 'Correlation between EMI and Loan Amount'; plot EMI*Loan_Amount / group=Delinquency grid markerattrs=(symbol=asterisk); run;

proc corr data=risk_data; run;

proc glm data = risk_data; class income_category; model fico_avg = income_category; run;

proc univariate data=risk_data normal; var _Numeric_; run;

data loan_data; set risk_data(keep=loan_amnt term int_rate installment grade tot_cur_bal mths_since_last_delinq home_ownership annual_inc verification_status purpose application_type dti delinq_2yrs out_prncp total_pymnt last_pymnt_amnt avg_cur_bal income_category fico_avg last_fico_avg Delinquency);
	if annual_inc>284000 then delete;
run;

proc stdize data=loan_data out=loan_data method=median reponly; var dti mths_since_last_delinq avg_cur_bal loan_inc_perc; run;

data loan_data; set loan_data; if not cmiss(of _numeric_); run;

proc reg data=loan_data(drop=purpose term grade income_category home_ownership application_type verification_status);
	model Delinquency = int_rate installment tot_cur_bal mths_since_last_delinq annual_inc dti delinq_2yrs out_prncp total_pymnt last_pymnt_amnt avg_cur_bal fico_avg last_fico_avg / vif;
run;

proc logistic data=loan_data plots(only)=roc;
	class purpose term grade income_category home_ownership application_type verification_status/param=reference;
	model Delinquency(event='1') = term int_rate installment grade tot_cur_bal mths_since_last_delinq home_ownership annual_inc verification_status purpose application_type dti delinq_2yrs out_prncp total_pymnt last_pymnt_amnt avg_cur_bal income_category fico_avg last_fico_avg/selection=stepwise rsq lackfit;
	output out=estimated predicted=estprob l=lower95 u=upper95;
run;
