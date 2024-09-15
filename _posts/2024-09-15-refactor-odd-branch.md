---
layout:       post
title:        "重构实践（二）：尽早返回 if 语句"
author:       "Quinlan"
header-style: text
catalog:      true
tags:
    - java 重构
---


# 重构实践（二）：尽早返回 if 语句

```java
    public String getPhone(Integer partnerId) {
        String phone = null;
        MtaPartnerContactsModel mtaPartnerContactsModel = mtaPartnerContactService.getOwnerContactInfo(partnerId);

        // 甲方联系人
        if (mtaPartnerContactsModel != null && mtaPartnerContactsModel.getMtaPartnerContactModelList() != null
                && mtaPartnerContactsModel.getMtaPartnerContactModelList().size() == 1) {
            phone = mtaPartnerContactService.getPhone(mtaPartnerContactsModel.getMtaPartnerContactModelList().get(0));
        }

        // 审核通过的数据在平台
        if (Strings.isNullOrEmpty(phone)) {
            // 获取商家手机号
            Partner partner = pmcRpcPartnerService.getById(partnerId);
            if (Objects.nonNull(partner)) {
                if (!partner.getExtendInfo().isEmpty()) {
                    phone = partner.getExtendInfo().get(PmcExtendAttribute.PARTNER_CERT_CONTACT_PHONE.getKeyInPMC());
                }
            }
        }

        // 未审核的数据在本地
        if (Strings.isNullOrEmpty(phone)) {
            // 获取商家手机号
            Map<String, String> attMap = mtaPartnerExtService.getProcessData(partnerId, PmcExtendAttribute.PARTNER_CERT_CONTACT_PHONE);
            if (!attMap.isEmpty()) {
                phone = attMap.get(PmcExtendAttribute.PARTNER_CERT_CONTACT_PHONE.getKeyInPMC());
            }
        }

        return phone;
    }
```

这段代码是获取客户的联系方式，由于数据的状态不同，存在多个数据源可以去兜底获取，所以这里代码首先去从 service 层获取联系人对象，然后检查数据是否有效。随后连续检查数据是否已经获取，通过对象是否为空来判断数据是否正常，如果为空，则尝试逐步从兜底数据源进行获取。

这里存在的问题在于 `Strings.isNullOrEmpty(phone)` 配合注释，不容易理解这里的检查目的，这里类似于编程语言中的或逻辑， `cond1 || cond2 || cond3`，在 cond1 满足的情况下，是不会去计算 cond2 和 cond3 的值，特别是其是函数表达式的返回值时，这样代码可以运行更高效。

这里也是同样的情况，数据源存在优先级，但这里通过检查 `phone` 是否有效的方式，比较难以快速理解这里的逻辑。

```java
    public String getPhone(Integer partnerId) {
        String phone = null;
        MtaPartnerContactsModel mtaPartnerContactsModel = mtaPartnerContactService.getOwnerContactInfo(partnerId);

        // 甲方联系人
        if (mtaPartnerContactsModel != null && mtaPartnerContactsModel.getMtaPartnerContactModelList() != null
                && mtaPartnerContactsModel.getMtaPartnerContactModelList().size() == 1) {
            phone = mtaPartnerContactService.getPhone(mtaPartnerContactsModel.getMtaPartnerContactModelList().get(0));
            if (!Strings.isNullOrEmpty(phone)) {
                return phone;
            }
        }

        // 审核通过的数据在平台
        Optional<String> phoneFromPlatform = getPhoneFromPlatform(partnerId);
        if (phoneFromPlatform.isPresent()) {
            return phoneFromPlatform.get();
        }

        // 未审核的数据在本地
        Optional<String> phoneFromLocalDB = getPhoneFromLocalDB(partnerId);
        if (phoneFromLocalDB.isPresent()) {
            return phoneFromLocalDB.get();
        }

        return phone;
    }

    private Optional<String> getPhoneFromPlatform(Integer partnerId) {
        Partner partner = pmcRpcPartnerService.getById(partnerId);
        if (Objects.nonNull(partner)) {
            if (!partner.getExtendInfo().isEmpty()) {
                String phone = partner.getExtendInfo().get(PmcExtendAttribute.PARTNER_CERT_CONTACT_PHONE.getKeyInPMC());
                return Optional.ofNullable(phone);
            }
        }

        return Optional.empty();
    }

    private Optional<String> getPhoneFromLocalDB(Integer partnerId) {
        // 获取商家手机号
        Map<String, String> attMap = mtaPartnerExtService.getProcessData(partnerId, PmcExtendAttribute.PARTNER_CERT_CONTACT_PHONE);
        if (!attMap.isEmpty()) {
            String phone = attMap.get(PmcExtendAttribute.PARTNER_CERT_CONTACT_PHONE.getKeyInPMC());
            return Optional.ofNullable(phone);
        }
        return Optional.empty();
    }
```

通过将代码块抽象为函数，同时配合 `Optional` 结构体，我们很容易表达我们获取数据的来源，以及可能获取不到的情况。因此这里的语意更加清晰。同时配合 `isPresent` 的返回表达式，我们能够快速短路逻辑，当获取到合法数据后快速返回。这样不仅将代码语意清晰化，同时也降低了函数的复杂度，从 11 降到了 6。

> 代码可以从我的 github 仓库获取，欢迎大家关注 star。[仓库链接](https://github.com/Cambrainnnnn/java-refactor/tree/main/src/main/java/refactor/oddIfBranch)
> 关于重构的原则，参考《重构：改善既有代码的设计》，其中“以卫语句取代嵌套条件表达式”