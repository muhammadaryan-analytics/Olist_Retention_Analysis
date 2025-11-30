<p align="center">
  <img src="https://github.com/user-attachments/assets/a58347ae-6a9d-4065-a757-b3d89b92c97d"  width="300"  alt="olistlogo">
</p>

# **Olist Marketplace Retention Analysis**
### **Why 97% of First-Time Buyers Never Return**
## Introduction

### **Who is Olist?**

Olist is a Brazilian e-commerce platform that connects small and medium-sized sellers to large online marketplaces like Mercado Livre. The platform provides tools for order management, logistics integration, payments, and customer support, helping sellers reach customers across Brazil without managing their own infrastructure.

### **The Business Problem**

Between 2016-2018, Olist experienced strong order volume growth (40% YoY) but faced a critical retention crisis: **96.9% of customers made only one purchase and never returned.**   

With rising customer acquisition costs, the growth team was spending heavily to replace churning customers rather than building a loyal buyer base. Leadership needed to understand whether this was fixable through operational improvements or a structural characteristic of their market position.


### **Core Question** 
Are customers leaving because of Olist's operational problems, or is this how Brazilian e-commerce worked in 2016-2018?

**If operational:** Identify the biggest controllable factor (delivery, seller quality, product issues), prove causation, and estimate the value of fixing it.   
**If structural:** Accept that retention will remain low and optimize acquisition efficiency instead.





### **Dataset Overview**
**Source:** Brazilian E-Commerce Public Dataset by Olist (**Kaggle**)      
[View Dataset](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce?resource=download)


### **Key Metrics Analyzed:**

- Customer retention and repeat purchase rates
- Delivery performance (on-time vs late)
- Review scores and customer satisfaction
- Seller quality indicators
- Product category mix
- Order values and geographic distribution


# **Executive Summary**
### **The Finding**         
Olist's 97% churn rate is not caused by operational failures. Statistical testing confirmed that even customers with perfect experiences—on-time delivery and 5-star reviews—only returned 3.2% of the time. This is a structural market behavior issue, not a fixable operations problem.

### **Evidence**

- **Delivery performance:** Late delivery (2.57% repeat rate) vs on-time delivery (3.24% repeat rate) shows only 0.67 percentage point difference (χ² = 8.32, p = 0.004)          
- **Customer satisfaction:** Review scores show no relationship with retention—negative reviews (3.10%), neutral (3.18%), and positive (3.18%) all yield identical repeat rates (χ² = 0.23, p = 0.889)       
- **Category variation:** Even naturally repeatable categories (pet supplies, health/beauty) show only 4-5% repeat rates vs 25-40% industry benchmarks       
- **Price and geography:** Order value and customer location show minimal variation in retention (all ~3%)























