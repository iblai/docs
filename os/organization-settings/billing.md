# Organization Settings: Billing

![Billing page showing the Premium plan, available credits with an Add Credits button, and Auto Recharge settings with threshold, recharge amount, and spending limit](/images/docs/os/organization-settings/organization_settings_billing.webp)

## Overview

The Billing page manages the organization's subscription plan and credit balance. The header reads "Billing — Manage your billing and subscription," and the page is organized into three cards: Plan, Credits, and Auto Recharge.

OS meters usage in credits rather than per-seat licenses: the balance is drawn down as the organization consumes the platform, and it can be topped up manually or automatically. Payments run through Stripe using the payment method on file.

To reach it, switch the top-bar toggle to **Admin**, open the settings dialog, and select **Billing** in the left sidebar.

## Target Audience
**Administrator**

## Sections and Controls

#### Plan
Shows the current subscription tier with a **Current** badge — **Premium** in the screenshot; other states are **Free** and **Trial**. Free and Trial plans display an **Upgrade** button that starts a Stripe checkout; Premium plans show their renewal date, and trials show days remaining.

#### Credits — Available
The credit balance ("Track your available credits and usage"), e.g. "16,984 Credits — Credits remaining." When applicable the card also shows **Consumed** (credits used this period) and **Resets on** (the next reset date).

#### Add Credits
Opens a dialog to top up manually: enter an **Amount (USD)** and confirm. The dialog notes "Your payment method on file will be charged for this amount." If no payment method exists yet, the button is replaced by **Manage Billing**, which opens the Stripe portal to add one.

#### Auto Recharge
"Top up your balance automatically when credits run low." The card shows an **Enabled**/**Disabled** badge and, when enabled, three stats: **Threshold** (balance at which a recharge triggers, `$0.00` in the screenshot), **Recharge Amount** (how much is added, `$10.00`), and **Spending Limit** (a cap on automatic spending — **Unlimited** when no cap is set).

#### Manage Usage
Opens the auto-recharge dialog where you toggle the feature and set the threshold, recharge amount, and spending limit (with an Unlimited option). Enabling with empty values applies sensible defaults ($5 threshold, $16 recharge). Auto recharge is available on paid plans once a payment method is on file.

## How to Use

#### Step 1: Open Billing
In **Admin** mode, open the settings dialog and click **Billing** in the left sidebar.

#### Step 2: Check your plan
Confirm the plan card shows the expected tier. On Free or Trial, click **Upgrade** to move to Premium via Stripe checkout.

#### Step 3: Add a payment method
If prompted with **Manage Billing**, follow it to the Stripe portal and save a payment method. This unlocks Add Credits and Auto Recharge.

#### Step 4: Top up credits
Click **Add Credits**, enter a USD amount, and confirm. The charge goes to your payment method on file and the balance updates.

#### Step 5: Configure auto recharge
Click **Manage Usage**, enable auto recharge, and set the threshold, recharge amount, and an optional spending limit. Your balance now tops up automatically whenever it falls below the threshold, never exceeding the limit you set.
