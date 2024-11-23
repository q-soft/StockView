SELECT t1.ID, t1.DocumentID, t1.DocumentNo, Convert(Date, t1.DocumentDate) As DocumentDate, t1.DocumentDate As DocumentTime, t1.DocumentType, t1.StockType, t1.AccountID, t1.AccountName, t1.ProductID, t1.ProductCode, t1.ProductName, t1.UOMID, UnitMaster.UOM, t1.Description, t1.QtyIn, t1.InPrice, t1.QtyOut, 
                         t1.OutPrice, t1.StoreID, t1.StoreName, t1.Department, t1.QtyInUnits, t1.PackQtyIn, t1.PackQtyOut, t1.StockOnHand, t1.BatchNo AS BatchID, ItemBatches.BatchNo, t1.Company, t1.Location, t1.Project, t1.CreatedBy, t1.FiscalYear
FROM            (SELECT        ROW_NUMBER() OVER (ORDER BY T.TransDate DESC) AS ID, T.DocumentID, T.TransID AS DocumentNo, T.TransDate AS DocumentDate, T.DocumentType, T.StockType, T.AccountID, T .AccountName, T .ProductID, 
                         Products.ProductCode, Products.ProductName, ItemUnits.UOMID, T .BatchNo, T .Description, T .QtyIn, T .InPrice, T .QtyOut, T .OutPrice, T .StoreID, T .Department, Stores.StoreName, ItemUnits.QtyInUnits, 
                         T .QtyIn * ItemUnits.QtyInUnits AS PackQtyIn, T .QtyOut * ItemUnits.QtyInUnits AS PackQtyOut, SUM(T.QtyIn * ItemUnits.QtyInUnits - T .QtyOut * ItemUnits.QtyInUnits) OVER (PARTITION BY T .ProductID
ORDER BY T .TransDate) AS StockOnHand, T.Company, T.Location, T.Project, T.CreatedBy, T.FiscalYear
FROM            ItemUnits INNER JOIN
                             (SELECT   DocumentID, TransID, TransDate, AccountID, AccountName, ProductID, UOMID, Description, QtyIn, InPrice, QtyOut, OutPrice, Department, StoreID, DocumentType, StockType, BatchNo, Location, Company, Project, CreatedBy, FiscalYear
FROM      (SELECT   OpeningInventory.ID AS DocumentID, OpeningInventory.OpeningStockID AS TransID, OpeningStock.OpeningDate AS TransDate, InventoryAsset AS AccountID, '' AS AccountName, OpeningStock.ProductID, OpeningStock.UOMID, 
                                 'OPENING QUANTITY' AS Description, CASE WHEN Company.ManageQtyByPackLoose = 0 THEN OpeningQty ELSE TotalQty / ItemUnits.QtyInUnits END AS QtyIn, IIF(TotalQty > 0, Total / TotalQty, 0) AS InPrice, 0 AS QtyOut, 0 AS OutPrice, OpeningStock.Department, 
                                 OpeningStock.StoreID, 'Opening Inventory' AS DocumentType, 'IN' AS StockType, OpeningStock.BatchNo, OpeningStock.Location, OpeningStock.Company, OpeningStock.Project, OpeningStock.CreatedBy, OpeningStock.FiscalYear, 
                                 OpeningInventory.IsApproved, Employee.EnableApprovals
                 FROM      OpeningStock INNER JOIN
                                 Employee ON OpeningStock.CreatedBy = Employee.Oid INNER JOIN
                                 Company ON OpeningStock.Company = Company.CompanyID INNER JOIN
                                 ItemUnits ON OpeningStock.UOMID = ItemUnits.ID INNER JOIN
                                 Products ON OpeningStock.ProductID = Products.ProductID INNER JOIN
                                 OpeningInventory ON OpeningStock.OpeningStockID = OpeningInventory.ID
                 WHERE   (OpeningStock.GCRecord IS NULL) AND (OpeningStock.IsVoided = 0)) AS T
WHERE   (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 WHEN EnableApprovals = 0 AND IsApproved = 1 THEN 1 ELSE 0 END) OR
                (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 ELSE 0 END)
UNION ALL
SELECT   DocumentID, TransID, TransDate, AccountID, AccountName, ProductID, UOMID, Description, QtyIn, InPrice, QtyOut, OutPrice, Department, StoreID, DocumentType, StockType, BatchNo, Location, Company, Project, CreatedBy, FiscalYear
FROM      (SELECT   OpeningStock.ID AS DocumentID, OpeningStock.AutoStockID AS TransID, OpeningStock.OpeningDate AS TransDate, InventoryAsset AS AccountID, '' AS AccountName, OpeningStock.ProductID, OpeningStock.UOMID, 
                                 'OPENING QUANTITY' AS Description, CASE WHEN Company.ManageQtyByPackLoose = 0 THEN OpeningQty ELSE TotalQty / ItemUnits.QtyInUnits END AS QtyIn, IIF(TotalQty > 0, Total / TotalQty, 0) AS InPrice, 0 AS QtyOut, 0 AS OutPrice, OpeningStock.Department, 
                                 OpeningStock.StoreID, 'Opening Stock' AS DocumentType, 'IN' AS StockType, OpeningStock.BatchNo, OpeningStock.Location, OpeningStock.Company, OpeningStock.Project, OpeningStock.CreatedBy, OpeningStock.FiscalYear, 
                                 OpeningStock.IsApproved, Employee.EnableApprovals
                 FROM      OpeningStock INNER JOIN
                                 Employee ON OpeningStock.CreatedBy = Employee.Oid INNER JOIN
                                 Company ON OpeningStock.Company = Company.CompanyID INNER JOIN
                                 ItemUnits ON OpeningStock.UOMID = ItemUnits.ID INNER JOIN
                                 Products ON OpeningStock.ProductID = Products.ProductID
                 WHERE   (OpeningStock.GCRecord IS NULL) AND (OpeningStock.IsVoided = 0) AND (OpeningStock.OpeningStockID IS NULL)) AS T
				 WHERE  (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 WHEN EnableApprovals = 0 AND IsApproved = 1 THEN 1 ELSE 0 END) OR
                  (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 ELSE 0 END)
UNION ALL
SELECT   GRNNo, TransID, TransDate, Supplier, Name, ProductID, UOMID, Name + ' ' + Description As Description, QtyIn, InPrice, QtyOut, OutPrice, Department, StoreID, DocumentType, StockType, BatchNo, Location, Company, Project, CreatedBy, FiscalYear
FROM      (SELECT   GRN.GRNNo, GRN.AutoGRNNo AS TransID, GRN.GRNDate AS TransDate, GRN.Supplier, GLAccount.Name, GRNDetails.ProductID, GRNDetails.UOMID, GRNDetails.Description, 
                                 CASE WHEN Company.ManageQtyByPackLoose = 0 THEN GRNDetails.Qty + GRNDetails.FreeQty ELSE GRNDetails.TotalQty / ItemUnits.QtyInUnits END AS QtyIn, CASE WHEN ISNULL(PurchaseInvoiceDetails.GRNo, '') = '' OR
                                 GRNDetails.Price = 0 THEN GRNDetails.Price ELSE ISNULL(CASE WHEN PurchaseInvoiceDetails.NetPrice > 0 THEN PurchaseInvoiceDetails.NetPrice ELSE CASE WHEN (PurchaseInvoiceDetails.LineTotal - GST - FixedTax - AdditionalTax) > 0 AND 
                                 (PurchaseInvoiceDetails.TotalQty) > 0 THEN ((PurchaseInvoiceDetails.LineTotal - GST - FixedTax - AdditionalTax) - (PurchaseInvoiceDetails.LineTotal - GST - FixedTax - AdditionalTax) * PurchaseInvoiceDetails.MasterDiscount / 100) / (PurchaseInvoiceDetails.TotalQty) 
                                 * ItemUnits.QtyInUnits ELSE 0 END END, 0) END AS InPrice, 0 AS QtyOut, 0 AS OutPrice, GRN.Department, DocumentBase.StoreID, 'Goods Received' AS DocumentType, 'IN' AS StockType, GRNDetails.BatchNo, GRN.Location, 
                                 GRN.Company, GRN.Project, GRN.CreatedBy, GRN.FiscalYear, GRN.IsApproved, Employee.EnableApprovals
                 FROM      GRNDetails INNER JOIN
                                 GRN ON GRNDetails.GRNNo = GRN.GRNNo INNER JOIN
                                 GLAccount ON GRN.Supplier = GLAccount.GLAccountID INNER JOIN
                                 Employee ON GRN.CreatedBy = Employee.Oid INNER JOIN
                                 Company ON GRN.Company = Company.CompanyID INNER JOIN
                                 DocumentBase ON GRNDetails.DocumentBaseID = DocumentBase.DocumentBaseID INNER JOIN
                                 ItemUnits ON GRNDetails.UOMID = ItemUnits.ID LEFT OUTER JOIN
                                 PurchaseInvoiceDetails ON GRNDetails.UOMID = PurchaseInvoiceDetails.UOMID AND GRNDetails.ProductID = PurchaseInvoiceDetails.ProductID AND GRNDetails.DocumentBaseID = PurchaseInvoiceDetails.GRNo
                 WHERE   (GRN.GCRecord IS NULL) AND (GRN.IsVoided = 0) AND (GRN.Status = 0) AND (GRNDetails.ManageStock = 1)) AS T
WHERE   (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 WHEN EnableApprovals = 0 AND IsApproved = 1 THEN 1 ELSE 0 END) OR
                (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 ELSE 0 END)
UNION ALL
	SELECT   GRNNo, TransID, TransDate, FromAccount, Name, ProductID, UOMID, Name + ' ' + Description As Description, QtyIn, InPrice, QtyOut, OutPrice, Department, StoreID, DocumentType, StockType, BatchNo, Location, Company, Project, CreatedBy, FiscalYear
FROM      (SELECT   GRN.GRNNo, GRN.AutoGRNNo AS TransID, GRN.GRNDate AS TransDate, GRN.FromAccount, GLAccount.Name, GRNDetails.ProductID, GRNDetails.UOMID, GRNDetails.Description, 
                                 CASE WHEN Company.ManageQtyByPackLoose = 0 THEN GRNDetails.Qty + GRNDetails.FreeQty ELSE GRNDetails.TotalQty / ItemUnits.QtyInUnits END AS QtyIn, CASE WHEN ISNULL(PurchaseInvoiceDetails.GRNo, '') = '' OR
                                 GRNDetails.Price = 0 THEN GRNDetails.Price ELSE ISNULL(CASE WHEN PurchaseInvoiceDetails.NetPrice > 0 THEN PurchaseInvoiceDetails.NetPrice ELSE CASE WHEN (PurchaseInvoiceDetails.LineTotal - GST - FixedTax - AdditionalTax) > 0 AND 
                                 (PurchaseInvoiceDetails.TotalQty) > 0 THEN ((PurchaseInvoiceDetails.LineTotal - GST - FixedTax - AdditionalTax) - (PurchaseInvoiceDetails.LineTotal - GST - FixedTax - AdditionalTax) * PurchaseInvoiceDetails.MasterDiscount / 100) / (PurchaseInvoiceDetails.TotalQty) 
                                 * ItemUnits.QtyInUnits ELSE 0 END END, 0) END AS InPrice, 0 AS QtyOut, 0 AS OutPrice, GRN.Department, DocumentBase.StoreID, 'Goods Received' AS DocumentType, 'IN' AS StockType, GRNDetails.BatchNo, GRN.Location, 
                                 GRN.Company, GRN.Project, GRN.CreatedBy, GRN.FiscalYear, GRN.IsApproved, Employee.EnableApprovals
                 FROM      GRNDetails INNER JOIN
                                 GRN ON GRNDetails.GRNNo = GRN.GRNNo INNER JOIN
                                 GLAccount ON GRN.FromAccount = GLAccount.GLAccountID INNER JOIN
                                 Employee ON GRN.CreatedBy = Employee.Oid INNER JOIN
                                 Company ON GRN.Company = Company.CompanyID INNER JOIN
                                 DocumentBase ON GRNDetails.DocumentBaseID = DocumentBase.DocumentBaseID INNER JOIN
                                 ItemUnits ON GRNDetails.UOMID = ItemUnits.ID LEFT OUTER JOIN
                                 PurchaseInvoiceDetails ON GRNDetails.UOMID = PurchaseInvoiceDetails.UOMID AND GRNDetails.ProductID = PurchaseInvoiceDetails.ProductID AND GRNDetails.DocumentBaseID = PurchaseInvoiceDetails.GRNo
                 WHERE   (GRN.GCRecord IS NULL) AND (GRN.IsVoided = 0) AND (GRN.Status = 0) AND (GRNDetails.ManageStock = 1) AND (Company.EnableARAPAccountsOnSalesPurchase = 1)) AS T
WHERE   (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 WHEN EnableApprovals = 0 AND IsApproved = 1 THEN 1 ELSE 0 END) OR
                (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 ELSE 0 END)
UNION ALL
SELECT   InvoiceNo, TransID, TransDate, Supplier, Name, ProductID, UOMID, Name + ' ' + Description As Description, QtyIn, InPrice, QtyOut, OutPrice, Department, StoreID, DocumentType, StockType, BatchNo, Location, Company, Project, CreatedBy, FiscalYear
FROM      (SELECT   PurchaseInvoice.InvoiceNo, PurchaseInvoice.AutoInvoiceNo AS TransID, PurchaseInvoice.InvoiceDate AS TransDate, PurchaseInvoice.Supplier, GLAccount_1.Name, PurchaseInvoiceDetails.ProductID, PurchaseInvoiceDetails.UOMID, 
                                 PurchaseInvoiceDetails.Description, 
                                 CASE WHEN purchaseinvoice.invoicetype = 0 THEN CASE WHEN Company.ManageQtyByPackLoose = 0 THEN PurchaseInvoiceDetails.Qty + PurchaseInvoiceDetails.FreeQty ELSE TotalQty / ItemUnits.QtyInUnits END ELSE 0 END AS QtyIn,
                                  CASE WHEN purchaseinvoice.invoicetype = 0 THEN CASE WHEN PurchaseInvoiceDetails.NetPrice > 0 THEN PurchaseInvoiceDetails.NetPrice ELSE CASE WHEN (LineTotal - GST - PurchaseInvoiceDetails.FixedTax - PurchaseInvoiceDetails.AdditionalTax) > 0 AND (TotalQty) > 0 THEN ((LineTotal - GST - PurchaseInvoiceDetails.FixedTax - PurchaseInvoiceDetails.AdditionalTax) 
                                 - (LineTotal - GST - PurchaseInvoiceDetails.FixedTax - PurchaseInvoiceDetails.AdditionalTax) * PurchaseInvoice.DiscountPercent / 100) / (TotalQty) * ItemUnits.QtyInUnits ELSE 0 END END ELSE 0 END AS InPrice, 
                                 CASE WHEN purchaseinvoice.invoicetype = 1 THEN CASE WHEN Company.ManageQtyByPackLoose = 0 THEN PurchaseInvoiceDetails.Qty + PurchaseInvoiceDetails.FreeQty ELSE TotalQty / ItemUnits.QtyInUnits END ELSE 0 END AS QtyOut,
                                  CASE WHEN purchaseinvoice.invoicetype = 1 THEN CASE WHEN PurchaseInvoiceDetails.NetPrice > 0 THEN PurchaseInvoiceDetails.NetPrice ELSE CASE WHEN (LineTotal - GST - PurchaseInvoiceDetails.FixedTax - PurchaseInvoiceDetails.AdditionalTax) > 0 AND (TotalQty) > 0 THEN (LineTotal - GST - PurchaseInvoiceDetails.FixedTax - PurchaseInvoiceDetails.AdditionalTax) 
                                 / (TotalQty) * ItemUnits.QtyInUnits ELSE 0 END END ELSE 0 END AS OutPrice, PurchaseInvoice.Department, DocumentBase.StoreID, 
                                 CASE WHEN purchaseinvoice.invoicetype = 0 THEN 'Purchase Invoice' ELSE 'Purchase Return' END AS DocumentType, CASE WHEN purchaseinvoice.invoicetype = 0 THEN 'IN' ELSE 'OUT' END AS StockType, 
                                 PurchaseInvoiceDetails.BatchNo, PurchaseInvoice.Location, PurchaseInvoice.Company, PurchaseInvoice.Project, PurchaseInvoice.CreatedBy, PurchaseInvoice.FiscalYear, PurchaseInvoice.IsApproved, 
                                 Employee.EnableApprovals
                 FROM      PurchaseInvoiceDetails INNER JOIN
                                 PurchaseInvoice ON PurchaseInvoiceDetails.InvoiceNo = PurchaseInvoice.InvoiceNo INNER JOIN
                                 GLAccount AS GLAccount_1 ON PurchaseInvoice.Supplier = GLAccount_1.GLAccountID INNER JOIN
                                 Employee ON PurchaseInvoice.CreatedBy = Employee.Oid INNER JOIN
                                 Company ON PurchaseInvoice.Company = Company.CompanyID INNER JOIN
                                 DocumentBase ON PurchaseInvoiceDetails.DocumentBaseID = DocumentBase.DocumentBaseID INNER JOIN
                                 ItemUnits ON PurchaseInvoiceDetails.UOMID = ItemUnits.ID
                 WHERE   (PurchaseInvoice.GRNNo IS NULL) AND (PurchaseInvoice.GCRecord IS NULL) AND (PurchaseInvoice.IsVoided = 0) AND (PurchaseInvoice.Status = 0) AND (PurchaseInvoiceDetails.ManageStock = 1)) AS T
WHERE   (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 WHEN EnableApprovals = 0 AND IsApproved = 1 THEN 1 ELSE 0 END) OR
                (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 ELSE 0 END)
UNION ALL
SELECT   InvoiceNo, TransID, TransDate, FromAccount, Name, ProductID, UOMID, Name + ' ' + Description As Description, QtyIn, InPrice, QtyOut, OutPrice, Department, StoreID, DocumentType, StockType, BatchNo, Location, Company, Project, CreatedBy, FiscalYear
FROM      (SELECT   PurchaseInvoice.InvoiceNo, PurchaseInvoice.AutoInvoiceNo AS TransID, PurchaseInvoice.InvoiceDate AS TransDate, PurchaseInvoice.FromAccount, GLAccount_1.Name, PurchaseInvoiceDetails.ProductID, PurchaseInvoiceDetails.UOMID, 
                                 PurchaseInvoiceDetails.Description, 
                                 CASE WHEN purchaseinvoice.invoicetype = 0 THEN CASE WHEN Company.ManageQtyByPackLoose = 0 THEN PurchaseInvoiceDetails.Qty + PurchaseInvoiceDetails.FreeQty ELSE TotalQty / ItemUnits.QtyInUnits END ELSE 0 END AS QtyIn,
                                  CASE WHEN purchaseinvoice.invoicetype = 0 THEN CASE WHEN PurchaseInvoiceDetails.NetPrice > 0 THEN PurchaseInvoiceDetails.NetPrice ELSE CASE WHEN (LineTotal - GST - PurchaseInvoiceDetails.FixedTax - PurchaseInvoiceDetails.AdditionalTax) > 0 AND (TotalQty) > 0 THEN ((LineTotal - GST - PurchaseInvoiceDetails.FixedTax - PurchaseInvoiceDetails.AdditionalTax) 
                                 - (LineTotal - GST - PurchaseInvoiceDetails.FixedTax - PurchaseInvoiceDetails.AdditionalTax) * PurchaseInvoice.DiscountPercent / 100) / (TotalQty) * ItemUnits.QtyInUnits ELSE 0 END END ELSE 0 END AS InPrice, 
                                 CASE WHEN purchaseinvoice.invoicetype = 1 THEN CASE WHEN Company.ManageQtyByPackLoose = 0 THEN PurchaseInvoiceDetails.Qty + PurchaseInvoiceDetails.FreeQty ELSE TotalQty / ItemUnits.QtyInUnits END ELSE 0 END AS QtyOut,
                                  CASE WHEN purchaseinvoice.invoicetype = 1 THEN CASE WHEN PurchaseInvoiceDetails.NetPrice > 0 THEN PurchaseInvoiceDetails.NetPrice ELSE CASE WHEN (LineTotal - GST - PurchaseInvoiceDetails.FixedTax - PurchaseInvoiceDetails.AdditionalTax) > 0 AND (TotalQty) > 0 THEN (LineTotal - GST - PurchaseInvoiceDetails.FixedTax - PurchaseInvoiceDetails.AdditionalTax) 
                                 / (TotalQty) * ItemUnits.QtyInUnits ELSE 0 END END ELSE 0 END AS OutPrice, PurchaseInvoice.Department, DocumentBase.StoreID, 
                                 CASE WHEN purchaseinvoice.invoicetype = 0 THEN 'Purchase Invoice' ELSE 'Purchase Return' END AS DocumentType, CASE WHEN purchaseinvoice.invoicetype = 0 THEN 'IN' ELSE 'OUT' END AS StockType, 
                                 PurchaseInvoiceDetails.BatchNo, PurchaseInvoice.Location, PurchaseInvoice.Company, PurchaseInvoice.Project, PurchaseInvoice.CreatedBy, PurchaseInvoice.FiscalYear, PurchaseInvoice.IsApproved, 
                                 Employee.EnableApprovals
                 FROM      PurchaseInvoiceDetails INNER JOIN
                                 PurchaseInvoice ON PurchaseInvoiceDetails.InvoiceNo = PurchaseInvoice.InvoiceNo INNER JOIN
                                 GLAccount AS GLAccount_1 ON PurchaseInvoice.FromAccount = GLAccount_1.GLAccountID INNER JOIN
                                 Employee ON PurchaseInvoice.CreatedBy = Employee.Oid INNER JOIN
                                 Company ON PurchaseInvoice.Company = Company.CompanyID INNER JOIN
                                 DocumentBase ON PurchaseInvoiceDetails.DocumentBaseID = DocumentBase.DocumentBaseID INNER JOIN
                                 ItemUnits ON PurchaseInvoiceDetails.UOMID = ItemUnits.ID
                 WHERE   (PurchaseInvoice.GRNNo IS NULL) AND (PurchaseInvoice.GCRecord IS NULL) AND (PurchaseInvoice.IsVoided = 0) AND (PurchaseInvoice.Status = 0) AND (PurchaseInvoiceDetails.ManageStock = 1) AND (Company.EnableARAPAccountsOnSalesPurchase = 1)) AS T
WHERE   (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 WHEN EnableApprovals = 0 AND IsApproved = 1 THEN 1 ELSE 0 END) OR
                (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 ELSE 0 END)
UNION ALL
SELECT   InvoiceNo, TransID, TransDate, Supplier, Name, PackageItem, UOMID, Name + ' ' + Description As Description, QtyIn, InPrice, QtyOut, OutPrice, Department, StoreID, DocumentType, StockType, Expr1, Location, Company, Project, CreatedBy, FiscalYear
FROM      (SELECT   PurchaseInvoice.InvoiceNo, PurchaseInvoice.AutoInvoiceNo AS TransID, PurchaseInvoice.InvoiceDate AS TransDate, PurchaseInvoice.Supplier, GLAccount_1.Name, ItemPackage.PackageItem, ItemPackage.UOMID, 
                                 ItemPackage.Description, CASE WHEN purchaseinvoice.invoicetype = 0 THEN ItemPackage.Qty ELSE 0 END AS QtyIn, CASE WHEN purchaseinvoice.invoicetype = 0 THEN ItemPackage.Price ELSE 0 END AS InPrice, 
                                 CASE WHEN purchaseinvoice.invoicetype = 1 THEN ItemPackage.Qty ELSE 0 END AS QtyOut, CASE WHEN purchaseinvoice.invoicetype = 1 THEN ItemPackage.Price ELSE 0 END AS OutPrice, PurchaseInvoice.Department, 
                                 DocumentBase.StoreID, CASE WHEN purchaseinvoice.invoicetype = 0 THEN 'Purchase Invoice' ELSE 'Purchase Return' END AS DocumentType, 
                                 CASE WHEN purchaseinvoice.invoicetype = 0 THEN 'IN' ELSE 'OUT' END AS StockType, NULL AS Expr1, PurchaseInvoice.Location, PurchaseInvoice.Company, PurchaseInvoice.Project, PurchaseInvoice.CreatedBy, 
                                 PurchaseInvoice.FiscalYear, PurchaseInvoice.IsApproved, Employee.EnableApprovals
                 FROM      PurchaseInvoiceDetails INNER JOIN
                                 PurchaseInvoice ON PurchaseInvoiceDetails.InvoiceNo = PurchaseInvoice.InvoiceNo INNER JOIN
                                 GLAccount AS GLAccount_1 ON PurchaseInvoice.Supplier = GLAccount_1.GLAccountID INNER JOIN
                                 Employee ON PurchaseInvoice.CreatedBy = Employee.Oid INNER JOIN
                                 Company ON PurchaseInvoice.Company = Company.CompanyID INNER JOIN
                                 DocumentBase ON PurchaseInvoiceDetails.DocumentBaseID = DocumentBase.DocumentBaseID INNER JOIN
                                 ItemPackage ON PurchaseInvoiceDetails.InvoiceNo = ItemPackage.PInvoiceNo AND PurchaseInvoiceDetails.ProductID = ItemPackage.ProductID INNER JOIN
                                 ItemUnits ON ItemPackage.UOMID = ItemUnits.ID
                 WHERE   (PurchaseInvoice.GCRecord IS NULL) AND (PurchaseInvoice.IsVoided = 0) AND (PurchaseInvoice.Status = 0) AND (PurchaseInvoiceDetails.ManageStock = 1)) AS T
WHERE   (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 WHEN EnableApprovals = 0 AND IsApproved = 1 THEN 1 ELSE 0 END) OR
                (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 ELSE 0 END)
UNION ALL
SELECT   InvoiceNo, TransID, TransDate, FromAccount, Name, PackageItem, UOMID, Name + ' ' + Description As Description, QtyIn, InPrice, QtyOut, OutPrice, Department, StoreID, DocumentType, StockType, Expr1, Location, Company, Project, CreatedBy, FiscalYear
FROM      (SELECT   PurchaseInvoice.InvoiceNo, PurchaseInvoice.AutoInvoiceNo AS TransID, PurchaseInvoice.InvoiceDate AS TransDate, PurchaseInvoice.FromAccount, GLAccount_1.Name, ItemPackage.PackageItem, ItemPackage.UOMID, 
                                 ItemPackage.Description, CASE WHEN purchaseinvoice.invoicetype = 0 THEN ItemPackage.Qty ELSE 0 END AS QtyIn, CASE WHEN purchaseinvoice.invoicetype = 0 THEN ItemPackage.Price ELSE 0 END AS InPrice, 
                                 CASE WHEN purchaseinvoice.invoicetype = 1 THEN ItemPackage.Qty ELSE 0 END AS QtyOut, CASE WHEN purchaseinvoice.invoicetype = 1 THEN ItemPackage.Price ELSE 0 END AS OutPrice, PurchaseInvoice.Department, 
                                 DocumentBase.StoreID, CASE WHEN purchaseinvoice.invoicetype = 0 THEN 'Purchase Invoice' ELSE 'Purchase Return' END AS DocumentType, 
                                 CASE WHEN purchaseinvoice.invoicetype = 0 THEN 'IN' ELSE 'OUT' END AS StockType, NULL AS Expr1, PurchaseInvoice.Location, PurchaseInvoice.Company, PurchaseInvoice.Project, PurchaseInvoice.CreatedBy, 
                                 PurchaseInvoice.FiscalYear, PurchaseInvoice.IsApproved, Employee.EnableApprovals
                 FROM      PurchaseInvoiceDetails INNER JOIN
                                 PurchaseInvoice ON PurchaseInvoiceDetails.InvoiceNo = PurchaseInvoice.InvoiceNo INNER JOIN
                                 GLAccount AS GLAccount_1 ON PurchaseInvoice.FromAccount = GLAccount_1.GLAccountID INNER JOIN
                                 Employee ON PurchaseInvoice.CreatedBy = Employee.Oid INNER JOIN
                                 Company ON PurchaseInvoice.Company = Company.CompanyID INNER JOIN
                                 DocumentBase ON PurchaseInvoiceDetails.DocumentBaseID = DocumentBase.DocumentBaseID INNER JOIN
                                 ItemPackage ON PurchaseInvoiceDetails.InvoiceNo = ItemPackage.PInvoiceNo AND PurchaseInvoiceDetails.ProductID = ItemPackage.ProductID INNER JOIN
                                 ItemUnits ON ItemPackage.UOMID = ItemUnits.ID
                 WHERE   (PurchaseInvoice.GCRecord IS NULL) AND (PurchaseInvoice.IsVoided = 0) AND (PurchaseInvoice.Status = 0) AND (PurchaseInvoiceDetails.ManageStock = 1) AND (Company.EnableARAPAccountsOnSalesPurchase = 1)) AS T
WHERE   (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 WHEN EnableApprovals = 0 AND IsApproved = 1 THEN 1 ELSE 0 END) OR
                (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 ELSE 0 END)
UNION ALL
SELECT   DONo, TransID, TransDate, Customer, Name, ProductID, UOMID, Name + ' ' + Description As Description, QtyIn, InPrice, QtyOut, OutPrice, Department, StoreID, DocumentType, StockType, BatchNo, Location, Company, Project, CreatedBy, FiscalYear
FROM      (SELECT   DeliveryOrder.DONo, DeliveryOrder.AutoDONo AS TransID, DeliveryOrder.DODate AS TransDate, DeliveryOrder.Customer, GLAccount_2.Name, DeliveryOrderDetails.ProductID, DeliveryOrderDetails.UOMID, 
                                 DeliveryOrderDetails.Description, 0 AS QtyIn, 0 AS InPrice, 
                                 CASE WHEN Company.ManageQtyByPackLoose = 0 THEN DeliveryOrderDetails.Qty + DeliveryOrderDetails.FreeQty ELSE DeliveryOrderDetails.TotalQty / ItemUnits.QtyInUnits END AS QtyOut, 
                                 ISNULL(CASE WHEN SalesInvoiceDetails.NetPrice > 0 THEN SalesInvoiceDetails.NetPrice ELSE CASE WHEN (SalesInvoiceDetails.LineTotal - GST - FixedTax - AdditionalTax) > 0 AND (SalesInvoiceDetails.TotalQty) 
                                 > 0 THEN ((SalesInvoiceDetails.LineTotal - GST - FixedTax - AdditionalTax) - (SalesInvoiceDetails.LineTotal - GST - FixedTax - AdditionalTax) * SalesInvoiceDetails.MasterDiscount / 100) / (SalesInvoiceDetails.TotalQty) * ItemUnits.QtyInUnits ELSE 0 END END, 0) AS OutPrice, 
                                 DeliveryOrder.Department, DocumentBase.StoreID, 'Delivery Order' AS DocumentType, 'OUT' AS StockType, DeliveryOrderDetails.BatchNo, DeliveryOrder.Location, DeliveryOrder.Company, DeliveryOrder.Project, 
                                 DeliveryOrder.CreatedBy, DeliveryOrder.FiscalYear, DeliveryOrder.IsApproved, Employee.EnableApprovals
                 FROM      DeliveryOrderDetails INNER JOIN
                                 DeliveryOrder ON DeliveryOrderDetails.DONo = DeliveryOrder.DONo INNER JOIN
                                 GLAccount AS GLAccount_2 ON DeliveryOrder.Customer = GLAccount_2.GLAccountID INNER JOIN
                                 Employee ON DeliveryOrder.CreatedBy = Employee.Oid INNER JOIN
                                 Company ON DeliveryOrder.Company = Company.CompanyID INNER JOIN
                                 DocumentBase ON DeliveryOrderDetails.DocumentBaseID = DocumentBase.DocumentBaseID INNER JOIN
                                 ItemUnits ON DeliveryOrderDetails.UOMID = ItemUnits.ID LEFT OUTER JOIN
                                 SalesInvoiceDetails ON DeliveryOrderDetails.DocumentBaseID = SalesInvoiceDetails.DONo AND DeliveryOrderDetails.ProductID = SalesInvoiceDetails.ProductID AND 
                                 DeliveryOrderDetails.UOMID = SalesInvoiceDetails.UOMID
                 WHERE   (DeliveryOrder.GCRecord IS NULL) AND (DeliveryOrder.IsVoided = 0) AND (DeliveryOrder.Status = 0) AND (DeliveryOrderDetails.ManageStock = 1)) AS T
WHERE   (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 WHEN EnableApprovals = 0 AND IsApproved = 1 THEN 1 ELSE 0 END) OR
                (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 ELSE 0 END)
UNION ALL
SELECT   DONo, TransID, TransDate, ToAccount, Name, ProductID, UOMID, Name + ' ' + Description As Description, QtyIn, InPrice, QtyOut, OutPrice, Department, StoreID, DocumentType, StockType, BatchNo, Location, Company, Project, CreatedBy, FiscalYear
FROM      (SELECT   DeliveryOrder.DONo, DeliveryOrder.AutoDONo AS TransID, DeliveryOrder.DODate AS TransDate, DeliveryOrder.ToAccount, GLAccount_2.Name, DeliveryOrderDetails.ProductID, DeliveryOrderDetails.UOMID, 
                                 DeliveryOrderDetails.Description, 0 AS QtyIn, 0 AS InPrice, 
                                 CASE WHEN Company.ManageQtyByPackLoose = 0 THEN DeliveryOrderDetails.Qty + DeliveryOrderDetails.FreeQty ELSE DeliveryOrderDetails.TotalQty / ItemUnits.QtyInUnits END AS QtyOut, 
                                 ISNULL(CASE WHEN SalesInvoiceDetails.NetPrice > 0 THEN SalesInvoiceDetails.NetPrice ELSE CASE WHEN (SalesInvoiceDetails.LineTotal - GST - FixedTax - AdditionalTax) > 0 AND (SalesInvoiceDetails.TotalQty) 
                                 > 0 THEN ((SalesInvoiceDetails.LineTotal - GST - FixedTax - AdditionalTax) - (SalesInvoiceDetails.LineTotal - GST - FixedTax - AdditionalTax) * SalesInvoiceDetails.MasterDiscount / 100) / (SalesInvoiceDetails.TotalQty) * ItemUnits.QtyInUnits ELSE 0 END END, 0) AS OutPrice, 
                                 DeliveryOrder.Department, DocumentBase.StoreID, 'Delivery Order' AS DocumentType, 'OUT' AS StockType, DeliveryOrderDetails.BatchNo, DeliveryOrder.Location, DeliveryOrder.Company, DeliveryOrder.Project, 
                                 DeliveryOrder.CreatedBy, DeliveryOrder.FiscalYear, DeliveryOrder.IsApproved, Employee.EnableApprovals
                 FROM      DeliveryOrderDetails INNER JOIN
                                 DeliveryOrder ON DeliveryOrderDetails.DONo = DeliveryOrder.DONo INNER JOIN
                                 GLAccount AS GLAccount_2 ON DeliveryOrder.ToAccount = GLAccount_2.GLAccountID INNER JOIN
                                 Employee ON DeliveryOrder.CreatedBy = Employee.Oid INNER JOIN
                                 Company ON DeliveryOrder.Company = Company.CompanyID INNER JOIN
                                 DocumentBase ON DeliveryOrderDetails.DocumentBaseID = DocumentBase.DocumentBaseID INNER JOIN
                                 ItemUnits ON DeliveryOrderDetails.UOMID = ItemUnits.ID LEFT OUTER JOIN
                                 SalesInvoiceDetails ON DeliveryOrderDetails.DocumentBaseID = SalesInvoiceDetails.DONo AND DeliveryOrderDetails.ProductID = SalesInvoiceDetails.ProductID AND 
                                 DeliveryOrderDetails.UOMID = SalesInvoiceDetails.UOMID
                 WHERE   (DeliveryOrder.GCRecord IS NULL) AND (DeliveryOrder.IsVoided = 0) AND (DeliveryOrder.Status = 0) AND (DeliveryOrderDetails.ManageStock = 1) AND (Company.EnableARAPAccountsOnSalesPurchase = 1)) AS T
WHERE   (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 WHEN EnableApprovals = 0 AND IsApproved = 1 THEN 1 ELSE 0 END) OR
                (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 ELSE 0 END)
UNION ALL
SELECT   InvoiceNo, TransID, TransDate, CustomerID, Name, ProductID, UOMID, Name + ' ' + Description As Description, QtyIn, InPrice, QtyOut, OutPrice, Department, StoreID, DocumentType, StockType, BatchNo, Location, Company, Project, CreatedBy, FiscalYear
FROM      (SELECT   SalesInvoice.InvoiceNo, SalesInvoice.AutoInvoiceNo AS TransID, SalesInvoice.InvoiceDate AS TransDate, SalesInvoice.CustomerID, GLAccount_1.Name, SalesInvoiceDetails.ProductID, SalesInvoiceDetails.UOMID, 
                                 SalesInvoiceDetails.Description, 
                                 CASE WHEN SalesInvoice.invoicetype = 1 THEN CASE WHEN Company.ManageQtyByPackLoose = 0 THEN SalesInvoiceDetails.Qty + SalesInvoiceDetails.FreeQty ELSE SalesInvoiceDetails.TotalQty / ItemUnits.QtyInUnits END ELSE 0
                                  END AS QtyIn, CASE WHEN SalesInvoice.invoicetype = 1 THEN CASE WHEN SalesInvoiceDetails.NetPrice > 0 THEN SalesInvoiceDetails.NetPrice ELSE CASE WHEN (LineTotal - GST - SalesInvoiceDetails.FixedTax - SalesInvoiceDetails.AdditionalTax) > 0 AND TotalQty > 0 THEN (LineTotal - GST - SalesInvoiceDetails.FixedTax - SalesInvoiceDetails.AdditionalTax) 
                                 / (TotalQty) * ItemUnits.QtyInUnits ELSE 0 END END ELSE 0 END AS InPrice, 
                                 CASE WHEN SalesInvoice.invoicetype = 0 THEN CASE WHEN Company.ManageQtyByPackLoose = 0 THEN SalesInvoiceDetails.Qty + SalesInvoiceDetails.FreeQty ELSE SalesInvoiceDetails.TotalQty / ItemUnits.QtyInUnits END ELSE 0
                                  END AS QtyOut, CASE WHEN SalesInvoice.invoicetype = 0 THEN CASE WHEN SalesInvoiceDetails.NetPrice > 0 THEN SalesInvoiceDetails.NetPrice ELSE CASE WHEN (LineTotal - GST - SalesInvoiceDetails.FixedTax - SalesInvoiceDetails.AdditionalTax) > 0 AND TotalQty > 0 THEN (LineTotal - GST - SalesInvoiceDetails.FixedTax - SalesInvoiceDetails.AdditionalTax) 
                                 / (TotalQty) * ItemUnits.QtyInUnits ELSE 0 END END ELSE 0 END AS OutPrice, SalesInvoice.Department, DocumentBase.StoreID, 
                                 CASE WHEN SalesInvoice.invoicetype = 0 THEN 'Sales Invoice' ELSE 'Sales Return' END AS DocumentType, CASE WHEN SalesInvoice.invoicetype = 0 THEN 'OUT' ELSE 'IN' END AS StockType, SalesInvoiceDetails.BatchNo, 
                                 SalesInvoice.Location, SalesInvoice.Company, SalesInvoice.Project, SalesInvoice.CreatedBy, SalesInvoice.FiscalYear, SalesInvoice.IsApproved, Employee.EnableApprovals
                 FROM      SalesInvoiceDetails INNER JOIN
                                 SalesInvoice ON SalesInvoiceDetails.InvoiceNo = SalesInvoice.InvoiceNo INNER JOIN
                                 DocumentBase ON SalesInvoiceDetails.DocumentBaseID = DocumentBase.DocumentBaseID INNER JOIN
                                 GLAccount AS GLAccount_1 ON SalesInvoice.CustomerID = GLAccount_1.GLAccountID INNER JOIN
                                 Employee ON SalesInvoice.CreatedBy = Employee.Oid INNER JOIN
                                 Company ON SalesInvoice.Company = Company.CompanyID INNER JOIN
                                 ItemUnits ON SalesInvoiceDetails.UOMID = ItemUnits.ID
                 WHERE   (SalesInvoice.DONo IS NULL) AND (SalesInvoice.GCRecord IS NULL) AND (SalesInvoice.IsVoided = 0) AND (SalesInvoice.InEffective = 0) AND (SalesInvoice.Status = 0) AND (SalesInvoice.InvoiceType = 0) AND 
                                 (SalesInvoice.IsDOAfterInvoice = 0) AND (SalesInvoiceDetails.ManageStock = 1)) AS T
WHERE   (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 WHEN EnableApprovals = 0 AND IsApproved = 1 THEN 1 ELSE 0 END) OR
                (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 ELSE 0 END)
UNION ALL
SELECT   InvoiceNo, TransID, TransDate, ToAccount, Name, ProductID, UOMID, Name + ' ' + Description As Description, QtyIn, InPrice, QtyOut, OutPrice, Department, StoreID, DocumentType, StockType, BatchNo, Location, Company, Project, CreatedBy, FiscalYear
FROM      (SELECT   SalesInvoice.InvoiceNo, SalesInvoice.AutoInvoiceNo AS TransID, SalesInvoice.InvoiceDate AS TransDate, SalesInvoice.ToAccount, GLAccount_1.Name, SalesInvoiceDetails.ProductID, SalesInvoiceDetails.UOMID, 
                                 SalesInvoiceDetails.Description, 
                                 CASE WHEN SalesInvoice.invoicetype = 1 THEN CASE WHEN Company.ManageQtyByPackLoose = 0 THEN SalesInvoiceDetails.Qty + SalesInvoiceDetails.FreeQty ELSE SalesInvoiceDetails.TotalQty / ItemUnits.QtyInUnits END ELSE 0
                                  END AS QtyIn, CASE WHEN SalesInvoice.invoicetype = 1 THEN CASE WHEN SalesInvoiceDetails.NetPrice > 0 THEN SalesInvoiceDetails.NetPrice ELSE CASE WHEN (LineTotal - GST - SalesInvoiceDetails.FixedTax - SalesInvoiceDetails.AdditionalTax) > 0 AND TotalQty > 0 THEN (LineTotal - GST - SalesInvoiceDetails.FixedTax - SalesInvoiceDetails.AdditionalTax) 
                                 / (TotalQty) * ItemUnits.QtyInUnits ELSE 0 END END ELSE 0 END AS InPrice, 
                                 CASE WHEN SalesInvoice.invoicetype = 0 THEN CASE WHEN Company.ManageQtyByPackLoose = 0 THEN SalesInvoiceDetails.Qty + SalesInvoiceDetails.FreeQty ELSE SalesInvoiceDetails.TotalQty / ItemUnits.QtyInUnits END ELSE 0
                                  END AS QtyOut, CASE WHEN SalesInvoice.invoicetype = 0 THEN CASE WHEN SalesInvoiceDetails.NetPrice > 0 THEN SalesInvoiceDetails.NetPrice ELSE CASE WHEN (LineTotal - GST - SalesInvoiceDetails.FixedTax - SalesInvoiceDetails.AdditionalTax) > 0 AND TotalQty > 0 THEN (LineTotal - GST - SalesInvoiceDetails.FixedTax - SalesInvoiceDetails.AdditionalTax) 
                                 / (TotalQty) * ItemUnits.QtyInUnits ELSE 0 END END ELSE 0 END AS OutPrice, SalesInvoice.Department, DocumentBase.StoreID, 
                                 CASE WHEN SalesInvoice.invoicetype = 0 THEN 'Sales Invoice' ELSE 'Sales Return' END AS DocumentType, CASE WHEN SalesInvoice.invoicetype = 0 THEN 'OUT' ELSE 'IN' END AS StockType, SalesInvoiceDetails.BatchNo, 
                                 SalesInvoice.Location, SalesInvoice.Company, SalesInvoice.Project, SalesInvoice.CreatedBy, SalesInvoice.FiscalYear, SalesInvoice.IsApproved, Employee.EnableApprovals
                 FROM      SalesInvoiceDetails INNER JOIN
                                 SalesInvoice ON SalesInvoiceDetails.InvoiceNo = SalesInvoice.InvoiceNo INNER JOIN
                                 DocumentBase ON SalesInvoiceDetails.DocumentBaseID = DocumentBase.DocumentBaseID INNER JOIN
                                 GLAccount AS GLAccount_1 ON SalesInvoice.ToAccount = GLAccount_1.GLAccountID INNER JOIN
                                 Employee ON SalesInvoice.CreatedBy = Employee.Oid INNER JOIN
                                 Company ON SalesInvoice.Company = Company.CompanyID INNER JOIN
                                 ItemUnits ON SalesInvoiceDetails.UOMID = ItemUnits.ID
                 WHERE   (SalesInvoice.DONo IS NULL) AND (SalesInvoice.GCRecord IS NULL) AND (SalesInvoice.IsVoided = 0) AND (SalesInvoice.InEffective = 0) AND (SalesInvoice.Status = 0) AND (SalesInvoice.InvoiceType = 0) AND 
                                 (SalesInvoice.IsDOAfterInvoice = 0) AND (SalesInvoiceDetails.ManageStock = 1) AND (Company.EnableARAPAccountsOnSalesPurchase = 1)) AS T
WHERE   (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 WHEN EnableApprovals = 0 AND IsApproved = 1 THEN 1 ELSE 0 END) OR
                (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 ELSE 0 END)
UNION ALL
SELECT   InvoiceNo, TransID, TransDate, CustomerID, Name, ProductID, UOMID, Name + ' ' + Description As Description, QtyIn, InPrice, QtyOut, OutPrice, Department, StoreID, DocumentType, StockType, BatchNo, Location, Company, Project, CreatedBy, FiscalYear
FROM      (SELECT   SalesInvoice.InvoiceNo, SalesInvoice.AutoInvoiceNo AS TransID, SalesInvoice.InvoiceDate AS TransDate, SalesInvoice.CustomerID, GLAccount_1.Name, ItemSchemesDetails.ProductID, ItemSchemesDetails.UOMID, 
                                 SalesInvoice.Notes AS Description, 0 AS QtyIn, 0 AS InPrice, ItemSchemesDetails.Qty AS QtyOut, 0 AS OutPrice, SalesInvoice.Department, DocumentBase.StoreID, 
                                 CASE WHEN SalesInvoice.invoicetype = 0 THEN 'Sales Invoice' ELSE 'Sales Return' END AS DocumentType, CASE WHEN SalesInvoice.invoicetype = 0 THEN 'OUT' ELSE 'IN' END AS StockType, NULL AS BatchNo, 
                                 SalesInvoice.Location, SalesInvoice.Company, SalesInvoice.Project, SalesInvoice.CreatedBy, SalesInvoice.FiscalYear, SalesInvoice.IsApproved, Employee.EnableApprovals
                 FROM      ItemSchemesDetails INNER JOIN
                                 ItemSchemes ON ItemSchemesDetails.SchemeID = ItemSchemes.SchemeID INNER JOIN
                                 SalesInvoice ON ItemSchemesDetails.InvoiceNo = SalesInvoice.InvoiceNo INNER JOIN
                                 DocumentBase ON ItemSchemes.DocumentBaseID = DocumentBase.DocumentBaseID INNER JOIN
                                 GLAccount AS GLAccount_1 ON SalesInvoice.CustomerID = GLAccount_1.GLAccountID INNER JOIN
                                 Employee ON SalesInvoice.CreatedBy = Employee.Oid INNER JOIN
                                 Company ON SalesInvoice.Company = Company.CompanyID INNER JOIN
                                 ItemUnits ON ItemSchemesDetails.UOMID = ItemUnits.ID
                 WHERE   (SalesInvoice.DONo IS NULL) AND (SalesInvoice.GCRecord IS NULL) AND (SalesInvoice.IsVoided = 0) AND (SalesInvoice.InEffective = 0) AND (SalesInvoice.Status = 0) AND (SalesInvoice.InvoiceType = 0) AND 
                                 (SalesInvoice.IsDOAfterInvoice = 0)) AS T
WHERE   (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 WHEN EnableApprovals = 0 AND IsApproved = 1 THEN 1 ELSE 0 END) OR
                (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 ELSE 0 END)
UNION ALL
SELECT   InvoiceNo, TransID, TransDate, ToAccount, Name, ProductID, UOMID, Name + ' ' + Description As Description, QtyIn, InPrice, QtyOut, OutPrice, Department, StoreID, DocumentType, StockType, BatchNo, Location, Company, Project, CreatedBy, FiscalYear
FROM      (SELECT   SalesInvoice.InvoiceNo, SalesInvoice.AutoInvoiceNo AS TransID, SalesInvoice.InvoiceDate AS TransDate, SalesInvoice.ToAccount, GLAccount_1.Name, ItemSchemesDetails.ProductID, ItemSchemesDetails.UOMID, 
                                 SalesInvoice.Notes AS Description, 0 AS QtyIn, 0 AS InPrice, ItemSchemesDetails.Qty AS QtyOut, 0 AS OutPrice, SalesInvoice.Department, DocumentBase.StoreID, 
                                 CASE WHEN SalesInvoice.invoicetype = 0 THEN 'Sales Invoice' ELSE 'Sales Return' END AS DocumentType, CASE WHEN SalesInvoice.invoicetype = 0 THEN 'OUT' ELSE 'IN' END AS StockType, NULL AS BatchNo, 
                                 SalesInvoice.Location, SalesInvoice.Company, SalesInvoice.Project, SalesInvoice.CreatedBy, SalesInvoice.FiscalYear, SalesInvoice.IsApproved, Employee.EnableApprovals
                 FROM      ItemSchemesDetails INNER JOIN
                                 ItemSchemes ON ItemSchemesDetails.SchemeID = ItemSchemes.SchemeID INNER JOIN
                                 SalesInvoice ON ItemSchemesDetails.InvoiceNo = SalesInvoice.InvoiceNo INNER JOIN
                                 DocumentBase ON ItemSchemes.DocumentBaseID = DocumentBase.DocumentBaseID INNER JOIN
                                 GLAccount AS GLAccount_1 ON SalesInvoice.ToAccount = GLAccount_1.GLAccountID INNER JOIN
                                 Employee ON SalesInvoice.CreatedBy = Employee.Oid INNER JOIN
                                 Company ON SalesInvoice.Company = Company.CompanyID INNER JOIN
                                 ItemUnits ON ItemSchemesDetails.UOMID = ItemUnits.ID
                 WHERE   (SalesInvoice.DONo IS NULL) AND (SalesInvoice.GCRecord IS NULL) AND (SalesInvoice.IsVoided = 0) AND (SalesInvoice.InEffective = 0) AND (SalesInvoice.Status = 0) AND (SalesInvoice.InvoiceType = 0) AND 
                                 (SalesInvoice.IsDOAfterInvoice = 0) AND (Company.EnableARAPAccountsOnSalesPurchase = 1)) AS T
WHERE   (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 WHEN EnableApprovals = 0 AND IsApproved = 1 THEN 1 ELSE 0 END) OR
                (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 ELSE 0 END)
UNION ALL
SELECT   InvoiceNo, TransID, TransDate, CustomerID, Name, ProductID, UOMID, Name + ' ' + Description AS Description, QtyIn, InPrice, QtyOut, OutPrice, Department, StoreID, DocumentType, StockType, BatchNo, Location, Company, Project, CreatedBy, 
                FiscalYear
FROM      (SELECT   SalesInvoice.InvoiceNo, SalesInvoice.AutoInvoiceNo AS TransID, SalesInvoice.InvoiceDate AS TransDate, SalesInvoice.CustomerID, GLAccount_1.Name, SalesInvoiceDetails.ProductID, SalesInvoiceDetails.UOMID, 
                                 SalesInvoiceDetails.Description, 
                                 CASE WHEN SalesInvoice.invoicetype = 1 THEN CASE WHEN Company.ManageQtyByPackLoose = 0 THEN SalesInvoiceDetails.Qty + SalesInvoiceDetails.FreeQty ELSE SalesInvoiceDetails.TotalQty / ItemUnits.QtyInUnits END ELSE 0
                                  END AS QtyIn, CostPrice AS InPrice, 
                                 CASE WHEN SalesInvoice.invoicetype = 0 THEN CASE WHEN Company.ManageQtyByPackLoose = 0 THEN SalesInvoiceDetails.Qty + SalesInvoiceDetails.FreeQty ELSE SalesInvoiceDetails.TotalQty / ItemUnits.QtyInUnits END ELSE 0
                                  END AS QtyOut, 
                                 CASE WHEN SalesInvoice.invoicetype = 0 THEN CASE WHEN SalesInvoiceDetails.NetPrice > 0 THEN SalesInvoiceDetails.NetPrice ELSE CASE WHEN (LineTotal - GST - SalesInvoiceDetails.FixedTax - SalesInvoiceDetails.AdditionalTax)
                                  > 0 AND TotalQty > 0 THEN (LineTotal - GST - SalesInvoiceDetails.FixedTax - SalesInvoiceDetails.AdditionalTax) / (TotalQty) * ItemUnits.QtyInUnits ELSE 0 END END ELSE 0 END AS OutPrice, SalesInvoice.Department, 
                                 DocumentBase.StoreID, CASE WHEN SalesInvoice.invoicetype = 0 THEN 'Sales Invoice' ELSE 'Sales Return' END AS DocumentType, CASE WHEN SalesInvoice.invoicetype = 0 THEN 'OUT' ELSE 'IN' END AS StockType, 
                                 SalesInvoiceDetails.BatchNo, SalesInvoice.Location, SalesInvoice.Company, SalesInvoice.Project, SalesReturn.ReturnCategory, SalesInvoice.CreatedBy, SalesInvoice.FiscalYear, SalesInvoice.IsApproved, 
                                 Employee.EnableApprovals
                 FROM      SalesInvoiceDetails INNER JOIN
                                 SalesInvoice ON SalesInvoiceDetails.InvoiceNo = SalesInvoice.InvoiceNo INNER JOIN
                                 DocumentBase ON SalesInvoiceDetails.DocumentBaseID = DocumentBase.DocumentBaseID INNER JOIN
                                 GLAccount AS GLAccount_1 ON SalesInvoice.CustomerID = GLAccount_1.GLAccountID INNER JOIN
                                 Employee ON SalesInvoice.CreatedBy = Employee.Oid INNER JOIN
                                 Company ON SalesInvoice.Company = Company.CompanyID INNER JOIN
                                 SalesReturn ON SalesInvoice.InvoiceNo = SalesReturn.InvoiceNo INNER JOIN
                                 ItemUnits ON SalesInvoiceDetails.UOMID = ItemUnits.ID
                 WHERE   (SalesInvoice.DONo IS NULL) AND (SalesInvoice.GCRecord IS NULL) AND (SalesInvoice.IsVoided = 0) AND (SalesInvoice.InEffective = 0) AND (SalesInvoice.Status = 0) AND (SalesInvoice.InvoiceType = 1) AND 
                                 (SalesReturn.ReturnCategory = 0) AND (SalesInvoiceDetails.ManageStock = 1)) AS T
WHERE   (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 WHEN EnableApprovals = 0 AND IsApproved = 1 THEN 1 ELSE 0 END) OR
                (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 ELSE 0 END)
UNION ALL
SELECT   InvoiceNo, TransID, TransDate, ToAccount, Name, ProductID, UOMID, Name + ' ' + Description AS Description, QtyIn, InPrice, QtyOut, OutPrice, Department, StoreID, DocumentType, StockType, BatchNo, Location, Company, Project, CreatedBy, 
                FiscalYear
FROM      (SELECT   SalesInvoice.InvoiceNo, SalesInvoice.AutoInvoiceNo AS TransID, SalesInvoice.InvoiceDate AS TransDate, SalesInvoice.ToAccount, GLAccount_1.Name, SalesInvoiceDetails.ProductID, SalesInvoiceDetails.UOMID, 
                                 SalesInvoiceDetails.Description, 
                                 CASE WHEN SalesInvoice.invoicetype = 1 THEN CASE WHEN Company.ManageQtyByPackLoose = 0 THEN SalesInvoiceDetails.Qty + SalesInvoiceDetails.FreeQty ELSE SalesInvoiceDetails.TotalQty / ItemUnits.QtyInUnits END ELSE 0
                                  END AS QtyIn, CostPrice AS InPrice, 
                                 CASE WHEN SalesInvoice.invoicetype = 0 THEN CASE WHEN Company.ManageQtyByPackLoose = 0 THEN SalesInvoiceDetails.Qty + SalesInvoiceDetails.FreeQty ELSE SalesInvoiceDetails.TotalQty / ItemUnits.QtyInUnits END ELSE 0
                                  END AS QtyOut, 
                                 CASE WHEN SalesInvoice.invoicetype = 0 THEN CASE WHEN SalesInvoiceDetails.NetPrice > 0 THEN SalesInvoiceDetails.NetPrice ELSE CASE WHEN (LineTotal - GST - SalesInvoiceDetails.FixedTax - SalesInvoiceDetails.AdditionalTax)
                                  > 0 AND TotalQty > 0 THEN (LineTotal - GST - SalesInvoiceDetails.FixedTax - SalesInvoiceDetails.AdditionalTax) / (TotalQty) * ItemUnits.QtyInUnits ELSE 0 END END ELSE 0 END AS OutPrice, SalesInvoice.Department, 
                                 DocumentBase.StoreID, CASE WHEN SalesInvoice.invoicetype = 0 THEN 'Sales Invoice' ELSE 'Sales Return' END AS DocumentType, CASE WHEN SalesInvoice.invoicetype = 0 THEN 'OUT' ELSE 'IN' END AS StockType, 
                                 SalesInvoiceDetails.BatchNo, SalesInvoice.Location, SalesInvoice.Company, SalesInvoice.Project, SalesReturn.ReturnCategory, SalesInvoice.CreatedBy, SalesInvoice.FiscalYear, SalesInvoice.IsApproved, 
                                 Employee.EnableApprovals
                 FROM      SalesInvoiceDetails INNER JOIN
                                 SalesInvoice ON SalesInvoiceDetails.InvoiceNo = SalesInvoice.InvoiceNo INNER JOIN
                                 DocumentBase ON SalesInvoiceDetails.DocumentBaseID = DocumentBase.DocumentBaseID INNER JOIN
                                 GLAccount AS GLAccount_1 ON SalesInvoice.ToAccount = GLAccount_1.GLAccountID INNER JOIN
                                 Employee ON SalesInvoice.CreatedBy = Employee.Oid INNER JOIN
                                 Company ON SalesInvoice.Company = Company.CompanyID INNER JOIN
                                 SalesReturn ON SalesInvoice.InvoiceNo = SalesReturn.InvoiceNo INNER JOIN
                                 ItemUnits ON SalesInvoiceDetails.UOMID = ItemUnits.ID
                 WHERE   (SalesInvoice.DONo IS NULL) AND (SalesInvoice.GCRecord IS NULL) AND (SalesInvoice.IsVoided = 0) AND (SalesInvoice.InEffective = 0) AND (SalesInvoice.Status = 0) AND (SalesInvoice.InvoiceType = 1) AND 
                                 (SalesReturn.ReturnCategory = 0) AND (SalesInvoiceDetails.ManageStock = 1) AND (Company.EnableARAPAccountsOnSalesPurchase = 1)) AS T
WHERE   (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 WHEN EnableApprovals = 0 AND IsApproved = 1 THEN 1 ELSE 0 END) OR
                (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 ELSE 0 END)
UNION ALL
SELECT   InvoiceNo, TransID, TransDate, CustomerID, Name, PackageItem, UOMID, Name + ' ' + Description As Description, QtyIn, InPrice, QtyOut, OutPrice, Department, StoreID, DocumentType, StockType, Expr1, Location, Company, Project, CreatedBy, FiscalYear
FROM      (SELECT   SalesInvoice.InvoiceNo, SalesInvoice.AutoInvoiceNo AS TransID, SalesInvoice.InvoiceDate AS TransDate, SalesInvoice.CustomerID, GLAccount_1.Name, ItemPackage.PackageItem, ItemPackage.UOMID, ItemPackage.Description, 
                                 CASE WHEN SalesInvoice.invoicetype = 1 THEN ItemPackage.Qty ELSE 0 END AS QtyIn, CASE WHEN SalesInvoice.invoicetype = 1 THEN ItemPackage.Price ELSE 0 END AS InPrice, 
                                 CASE WHEN SalesInvoice.invoicetype = 0 THEN ItemPackage.Qty ELSE 0 END AS QtyOut, CASE WHEN SalesInvoice.invoicetype = 0 THEN ItemPackage.Price ELSE 0 END AS OutPrice, SalesInvoice.Department, 
                                 DocumentBase.StoreID, CASE WHEN SalesInvoice.invoicetype = 0 THEN 'Purchase Invoice' ELSE 'Purchase Return' END AS DocumentType, CASE WHEN SalesInvoice.invoicetype = 0 THEN 'OUT' ELSE 'IN' END AS StockType, NULL 
                                 AS Expr1, SalesInvoice.Location, SalesInvoice.Company, SalesInvoice.Project, SalesInvoice.CreatedBy, SalesInvoice.FiscalYear, SalesInvoice.IsApproved, Employee.EnableApprovals
                 FROM      SalesInvoiceDetails INNER JOIN
                                 SalesInvoice ON SalesInvoiceDetails.InvoiceNo = SalesInvoice.InvoiceNo INNER JOIN
                                 GLAccount AS GLAccount_1 ON SalesInvoice.CustomerID = GLAccount_1.GLAccountID INNER JOIN
                                 Employee ON SalesInvoice.CreatedBy = Employee.Oid INNER JOIN
                                 Company ON SalesInvoice.Company = Company.CompanyID INNER JOIN
                                 DocumentBase ON SalesInvoiceDetails.DocumentBaseID = DocumentBase.DocumentBaseID INNER JOIN
                                 ItemPackage ON SalesInvoiceDetails.InvoiceNo = ItemPackage.SInvoiceNo AND SalesInvoiceDetails.ProductID = ItemPackage.ProductID INNER JOIN
                                 ItemUnits ON ItemPackage.UOMID = ItemUnits.ID
                 WHERE   (SalesInvoice.GCRecord IS NULL) AND (SalesInvoice.IsVoided = 0) AND (SalesInvoice.Status = 0) AND (SalesInvoiceDetails.ManageStock = 1)) AS T
				 WHERE  (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 WHEN EnableApprovals = 0 AND IsApproved = 1 THEN 1 ELSE 0 END) OR
                  (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 ELSE 0 END)
UNION ALL
SELECT   InvoiceNo, TransID, TransDate, ToAccount, Name, PackageItem, UOMID, Name + ' ' + Description As Description, QtyIn, InPrice, QtyOut, OutPrice, Department, StoreID, DocumentType, StockType, Expr1, Location, Company, Project, CreatedBy, FiscalYear
FROM      (SELECT   SalesInvoice.InvoiceNo, SalesInvoice.AutoInvoiceNo AS TransID, SalesInvoice.InvoiceDate AS TransDate, SalesInvoice.ToAccount, GLAccount_1.Name, ItemPackage.PackageItem, ItemPackage.UOMID, ItemPackage.Description, 
                                 CASE WHEN SalesInvoice.invoicetype = 1 THEN ItemPackage.Qty ELSE 0 END AS QtyIn, CASE WHEN SalesInvoice.invoicetype = 1 THEN ItemPackage.Price ELSE 0 END AS InPrice, 
                                 CASE WHEN SalesInvoice.invoicetype = 0 THEN ItemPackage.Qty ELSE 0 END AS QtyOut, CASE WHEN SalesInvoice.invoicetype = 0 THEN ItemPackage.Price ELSE 0 END AS OutPrice, SalesInvoice.Department, 
                                 DocumentBase.StoreID, CASE WHEN SalesInvoice.invoicetype = 0 THEN 'Purchase Invoice' ELSE 'Purchase Return' END AS DocumentType, CASE WHEN SalesInvoice.invoicetype = 0 THEN 'OUT' ELSE 'IN' END AS StockType, NULL 
                                 AS Expr1, SalesInvoice.Location, SalesInvoice.Company, SalesInvoice.Project, SalesInvoice.CreatedBy, SalesInvoice.FiscalYear, SalesInvoice.IsApproved, Employee.EnableApprovals
                 FROM      SalesInvoiceDetails INNER JOIN
                                 SalesInvoice ON SalesInvoiceDetails.InvoiceNo = SalesInvoice.InvoiceNo INNER JOIN
                                 GLAccount AS GLAccount_1 ON SalesInvoice.ToAccount = GLAccount_1.GLAccountID INNER JOIN
                                 Employee ON SalesInvoice.CreatedBy = Employee.Oid INNER JOIN
                                 Company ON SalesInvoice.Company = Company.CompanyID INNER JOIN
                                 DocumentBase ON SalesInvoiceDetails.DocumentBaseID = DocumentBase.DocumentBaseID INNER JOIN
                                 ItemPackage ON SalesInvoiceDetails.InvoiceNo = ItemPackage.SInvoiceNo AND SalesInvoiceDetails.ProductID = ItemPackage.ProductID INNER JOIN
                                 ItemUnits ON ItemPackage.UOMID = ItemUnits.ID
                 WHERE   (SalesInvoice.GCRecord IS NULL) AND (SalesInvoice.IsVoided = 0) AND (SalesInvoice.Status = 0) AND (SalesInvoiceDetails.ManageStock = 1) AND (Company.EnableARAPAccountsOnSalesPurchase = 1)) AS T
				 WHERE  (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 WHEN EnableApprovals = 0 AND IsApproved = 1 THEN 1 ELSE 0 END) OR
                  (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 ELSE 0 END)
UNION ALL
SELECT   InvoiceNo, TransID, TransDate, CustomerID, Name, ProductID, UOMID, Name + ' ' + Description As Description, QtyIn, InPrice, QtyOut, OutPrice, Department, StoreID, DocumentType, StockType, BatchNo, Location, Company, Project, CreatedBy, FiscalYear
FROM      (SELECT   POS.InvoiceNo, POS.AutoInvoiceNo AS TransID, POS.InvoiceDate AS TransDate, POS.CustomerID, GLAccount_1.Name, POSDetails.ProductID, POSDetails.UOMID, POSDetails.Description, 
                                 CASE WHEN POSDetails.Qty < 0 THEN ABS(POSDetails.Qty + POSDetails.FreeQty) ELSE 0 END AS QtyIn, CASE WHEN POSDetails.Qty < 0 THEN (LineTotal - GST) / ABS(Qty + FreeQty) ELSE 0 END AS InPrice, 
                                 CASE WHEN Qty < 0 THEN 0 ELSE POSDetails.Qty + POSDetails.FreeQty END AS QtyOut, CASE WHEN POSDetails.Qty > 0 THEN (LineTotal - GST) / (Qty + FreeQty) ELSE 0 END AS OutPrice, POS.Department, DocumentBase.StoreID, 
                                 'Point Of Sales' AS DocumentType, CASE WHEN Qty > 0 THEN 'OUT' ELSE 'IN' END AS StockType, POSDetails.BatchNo, POS.Location, POS.Company, POS.Project, POS.CreatedBy, POS.FiscalYear, POS.IsApproved, 
                                 Employee.EnableApprovals
                 FROM      POSDetails INNER JOIN
                                 POS ON POSDetails.InvoiceNo = POS.InvoiceNo INNER JOIN
                                 DocumentBase ON POSDetails.DocumentBaseID = DocumentBase.DocumentBaseID LEFT OUTER JOIN
                                 GLAccount AS GLAccount_1 ON POS.CustomerID = GLAccount_1.GLAccountID INNER JOIN
                                 Employee ON POS.CreatedBy = Employee.Oid
                 WHERE   (POS.GCRecord IS NULL) AND (POS.IsVoided = 0) AND (POS.Status = 0)) AS T
WHERE   (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 WHEN EnableApprovals = 0 AND IsApproved = 1 THEN 1 ELSE 0 END) OR
                (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 ELSE 0 END)                    
UNION ALL
SELECT   StockAdjID, AutoAdjID, AdjustmentDate, AccountName, Expr1, ProductID, UOMID, Notes, QtyAdd, InPrice, QtyLess, OutPrice, Department, StoreID, DocumentType, StockType, BatchNo, Location, Company, Project, CreatedBy, FiscalYear
FROM      (SELECT   StockAdjustment.StockAdjID, StockAdjustment.AutoAdjID, StockAdjustment.AdjustmentDate, NULL AS AccountName, '' AS Expr1, StockAdjustmentDetails.ProductID, StockAdjustmentDetails.UOMID, StockAdjustment.Notes, 
                CASE WHEN Company.ManageQtyByPackLoose = 0 THEN StockAdjustmentDetails.QtyAdd ELSE CASE WHEN QtyAdd > 0 OR PackQtyAdd > 0 THEN TotalQty / ItemUnits.QtyInUnits ELSE 0 END END AS QtyAdd, StockAdjustmentDetails.Price AS InPrice, 
                CASE WHEN Company.ManageQtyByPackLoose = 0 THEN StockAdjustmentDetails.QtyLess ELSE CASE WHEN QtyLess > 0 OR PackQtyLess > 0 THEN TotalQty / ItemUnits.QtyInUnits ELSE 0 END END AS QtyLess, StockAdjustmentDetails.Price AS OutPrice, StockAdjustment.Department, 
                DocumentBase.StoreID, 'Stock Adjustment' AS DocumentType, CASE WHEN QtyAdd > 0 THEN 'IN' ELSE 'OUT' END AS StockType, StockAdjustmentDetails.BatchNo, StockAdjustment.Location, StockAdjustment.Company, StockAdjustment.Project, 
                StockAdjustment.CreatedBy, StockAdjustment.FiscalYear, StockAdjustment.IsApproved, Employee.EnableApprovals
FROM      StockAdjustment INNER JOIN
                StockAdjustmentDetails ON StockAdjustment.StockAdjID = StockAdjustmentDetails.StockAdjID INNER JOIN
                Employee ON StockAdjustment.CreatedBy = Employee.Oid INNER JOIN
                DocumentBase ON StockAdjustmentDetails.DocumentBaseID = DocumentBase.DocumentBaseID INNER JOIN
                Company ON StockAdjustment.Company = Company.CompanyID INNER JOIN
                ItemUnits ON StockAdjustmentDetails.UOMID = ItemUnits.ID
WHERE   (StockAdjustment.GCRecord IS NULL) AND (StockAdjustment.IsVoided = 0) AND (StockAdjustment.Status = 0)) AS T
WHERE   (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 WHEN EnableApprovals = 0 AND IsApproved = 1 THEN 1 ELSE 0 END) OR
                (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 ELSE 0 END)
UNION ALL
SELECT   TransferID, AutoTransferID, TransferDate, AccountName, Expr1, ProductID, UOMID, Expr2, QtyIn, InPrice, QtyOut, OutPrice, Department, ToStore, DocumentType, StockType, BatchNo, Location, Company, Project, CreatedBy, FiscalYear
FROM      (SELECT   StockTransfer.TransferID, StockTransfer.AutoTransferID, CASE WHEN StockTransfer.ReceiveDate IS NULL THEN StockTransfer.TransferDate ELSE StockTransfer.ReceiveDate END AS TransferDate, StockTransfer.AccountName, '' AS Expr1, 
                StockTransDetails.ProductID, StockTransDetails.UOMID, Stores_1.StoreName + N' ' + ISNULL(StockTransfer.Notes, '') AS Expr2, 
                CASE WHEN Company.ManageQtyByPackLoose = 0 THEN StockTransDetails.Qty ELSE StockTransDetails.TotalQty / ItemUnits.QtyInUnits END AS QtyIn, StockTransDetails.Price AS InPrice, 0 AS QtyOut, 0 AS OutPrice, StockTransfer.Department, 
                StockTransDetails.ToStore, 'Stock Transfer' AS DocumentType, 'IN' AS StockType, StockTransDetails.BatchNo, StockTransfer.Location, Stores.Company, StockTransfer.Project, StockTransfer.CreatedBy, StockTransfer.FiscalYear, 
                StockTransfer.IsApproved, Employee.EnableApprovals
FROM      StockTransfer INNER JOIN
                StockTransDetails ON StockTransfer.TransferID = StockTransDetails.TransferID INNER JOIN
                Employee ON StockTransfer.CreatedBy = Employee.Oid INNER JOIN
                Company ON StockTransfer.Company = Company.CompanyID INNER JOIN
                ItemUnits ON StockTransDetails.UOMID = ItemUnits.ID INNER JOIN
                Stores ON StockTransDetails.ToStore = Stores.StoreID INNER JOIN
                Stores AS Stores_1 ON StockTransDetails.FromStore = Stores_1.StoreID
WHERE   (StockTransfer.GCRecord IS NULL) AND (StockTransfer.IsVoided = 0) AND (StockTransfer.Status = 0) AND (StockTransfer.DocumentType IN (2, 3))) AS T
WHERE   (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 WHEN EnableApprovals = 0 AND IsApproved = 1 THEN 1 ELSE 0 END) OR
                (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 ELSE 0 END)
UNION ALL
SELECT   TransferID, AutoTransferID, TransferDate, AccountName, Expr1, ProductID, UOMID, Expr2, QtyIn, InPrice, QtyOut, OutPrice, Department, FromStore, DocumentType, StockType, BatchNo, Location, Company, Project, CreatedBy, FiscalYear
FROM      (SELECT   StockTransfer_1.TransferID, StockTransfer_1.AutoTransferID, StockTransfer_1.TransferDate, StockTransfer_1.AccountName, '' AS Expr1, StockTransDetails_1.ProductID, StockTransDetails_1.UOMID, 
                Stores_1.StoreName + N' ' + ISNULL(StockTransfer_1.Notes, '') AS Expr2, 0 AS QtyIn, 0 AS InPrice, 
                CASE WHEN Company.ManageQtyByPackLoose = 0 THEN StockTransDetails_1.Qty ELSE StockTransDetails_1.TotalQty / ItemUnits.QtyInUnits END AS QtyOut, StockTransDetails_1.Price AS OutPrice, StockTransfer_1.Department, 
                StockTransDetails_1.FromStore, 'Stock Transfer' AS DocumentType, 'OUT' AS StockType, StockTransDetails_1.BatchNo, StockTransfer_1.Location, Stores.Company, StockTransfer_1.Project, StockTransfer_1.CreatedBy, 
                StockTransfer_1.FiscalYear, StockTransfer_1.IsApproved, Employee.EnableApprovals
FROM      StockTransfer AS StockTransfer_1 INNER JOIN
                StockTransDetails AS StockTransDetails_1 ON StockTransfer_1.TransferID = StockTransDetails_1.TransferID INNER JOIN
                Employee ON StockTransfer_1.CreatedBy = Employee.Oid INNER JOIN
                ItemUnits ON StockTransDetails_1.UOMID = ItemUnits.ID INNER JOIN
                Stores ON StockTransDetails_1.FromStore = Stores.StoreID INNER JOIN
                Company ON Stores.Company = Company.CompanyID  INNER JOIN
                Stores AS Stores_1 ON StockTransDetails_1.ToStore = Stores_1.StoreID
WHERE   (StockTransfer_1.GCRecord IS NULL) AND (StockTransfer_1.IsVoided = 0) AND (StockTransfer_1.Status = 0) AND (StockTransfer_1.DocumentType <> 0)) AS T
WHERE   (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 WHEN EnableApprovals = 0 AND IsApproved = 1 THEN 1 ELSE 0 END) OR
                (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 ELSE 0 END)

UNION ALL
SELECT   IssueID, AutoIssueNo, IssueDate, AccountName, Expr1, ProductID, UOMID, Notes, QtyIn, InPrice, QtyOut, OutPrice, Department, StoreID, DocumentType, StockType, BatchNo, Location, Company, Project, CreatedBy, FiscalYear
FROM      (SELECT   StockIssue.IssueID, StockIssue.AutoIssueNo, StockIssue.IssueDate, NULL AS AccountName, '' AS Expr1, StockIssueDetail.ProductID, StockIssueDetail.UOMID, StockIssue.Notes, 0 AS QtyIn, 0 AS InPrice, 
                                 CASE WHEN Company.ManageQtyByPackLoose = 0 THEN Qty ELSE TotalQty / ItemUnits.QtyInUnits END AS QtyOut, StockIssueDetail.Price AS OutPrice, StockIssue.Department, DocumentBase.StoreID, 'Stock Issued' AS DocumentType, 'OUT' AS StockType, StockIssueDetail.BatchNo, StockIssue.Location, 
                                 StockIssue.Company, StockIssue.Project, StockIssue.CreatedBy, StockIssue.FiscalYear, StockIssue.IsApproved, Employee.EnableApprovals
                 FROM      StockIssue INNER JOIN
                                 StockIssueDetail ON StockIssue.IssueID = StockIssueDetail.IssueNo INNER JOIN
                                 Employee ON StockIssue.CreatedBy = Employee.Oid INNER JOIN 
                                 DocumentBase ON StockIssueDetail.DocumentBaseID = DocumentBase.DocumentBaseID 
INNER JOIN
                Company ON StockIssue.Company = Company.CompanyID INNER JOIN
                                 ItemUnits ON StockIssueDetail.UOMID = ItemUnits.ID
                 WHERE   (StockIssue.GCRecord IS NULL) AND (StockIssue.IsVoided = 0) AND (StockIssue.Status = 0)) AS T
WHERE   (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 WHEN EnableApprovals = 0 AND IsApproved = 1 THEN 1 ELSE 0 END) OR
                (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 ELSE 0 END)
UNION ALL
SELECT   ReceiveID, AutoReceivedNo, ReceivedDate, AccountName, Expr1, ProductID, UOMID, Notes, QtyIn, InPrice, QtyOut, OutPrice, Department, StoreID, DocumentType, StockType, BatchNo, Location, Company, Project, CreatedBy, FiscalYear
FROM      (SELECT   StockReceive.ReceiveID, StockReceive.AutoReceivedNo, StockReceive.ReceivedDate, NULL AS AccountName, '' AS Expr1, StockReceiveDetail.ProductID, StockReceiveDetail.UOMID, StockReceive.Notes, 
                                 CASE WHEN Company.ManageQtyByPackLoose = 0 THEN Qty ELSE TotalQty / ItemUnits.QtyInUnits END AS QtyIn, StockReceiveDetail.Price AS InPrice, 0 AS QtyOut, 0 AS OutPrice, StockReceive.Department, DocumentBase.StoreID, 'Stock Received' AS DocumentType, 'IN' AS StockType, 
                                 StockReceiveDetail.BatchNo, StockReceive.Location, StockReceive.Company, StockReceive.Project, StockReceive.CreatedBy, StockReceive.FiscalYear, StockReceive.IsApproved, Employee.EnableApprovals
                 FROM      StockReceive INNER JOIN
                                 StockReceiveDetail ON StockReceive.ReceiveID = StockReceiveDetail.ReceivedNo INNER JOIN
                                 Employee ON StockReceive.CreatedBy = Employee.Oid INNER JOIN
                                 DocumentBase ON StockReceiveDetail.DocumentBaseID = DocumentBase.DocumentBaseID
INNER JOIN
                Company ON StockReceive.Company = Company.CompanyID INNER JOIN
                                 ItemUnits ON StockReceiveDetail.UOMID = ItemUnits.ID
                 WHERE   (StockReceive.GCRecord IS NULL) AND (StockReceive.IsVoided = 0) AND (StockReceive.Status = 0)) AS T
				 WHERE  (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 WHEN EnableApprovals = 0 AND IsApproved = 1 THEN 1 ELSE 0 END) OR
                  (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 ELSE 0 END)
UNION ALL
SELECT   OrderNo, AutoOrderNo, OrderDate, AccountName, Expr1, BomProduct, UOMID, Remarks, QtyIn, InPrice, QtyOut, OutPrice, Department, StoreID, DocumentType, StockType, BatchNo, Location, Company, Project, CreatedBy, FiscalYear
FROM      (SELECT   WorkOrder.OrderNo, WorkOrder.AutoOrderNo, WorkOrder.OrderDate, WorkOrder.CustomerID AS AccountName, '' AS Expr1, WorkOrderDetails.BomProduct, WorkOrderDetails.UOMID, WorkOrder.Remarks, 0 AS QtyIn, 0 AS InPrice, 
                                 CASE WHEN Qty > 0 THEN Qty ELSE ISNULL(WorkOrderDetails.MaterialUsed, 0) END AS QtyOut, WorkOrderDetails.CostPrice AS OutPrice, WorkOrder.Department, DocumentBase.StoreID, 'Work Order' AS DocumentType, 'OUT' AS StockType, WorkOrder.BatchNo, 
                                 WorkOrder.Location, WorkOrder.Company, WorkOrder.Project, WorkOrder.CreatedBy, WorkOrder.FiscalYear, WorkOrder.IsApproved, Employee.EnableApprovals
                 FROM      WorkOrder INNER JOIN
                                 WorkOrderDetails ON WorkOrder.OrderNo = WorkOrderDetails.OrderNo INNER JOIN
                                 Employee ON WorkOrder.CreatedBy = Employee.Oid INNER JOIN
                                 DocumentBase ON WorkOrderDetails.DocumentBaseID = DocumentBase.DocumentBaseID
                 WHERE   (WorkOrder.GCRecord IS NULL) AND (WorkOrder.IsVoided = 0)) AS T
WHERE   (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 WHEN EnableApprovals = 0 AND IsApproved = 1 THEN 1 ELSE 0 END) OR
                (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 ELSE 0 END)
UNION ALL
SELECT   OrderNo, AutoOrderNo, OrderDate, AccountName, Expr1, WasteProduct, BaseUOM, Remarks, QtyIn, InPrice, QtyOut, OutPrice, Department, WastageStore, DocumentType, StockType, BatchNo, Location, Company, Project, CreatedBy, 
                FiscalYear
FROM      (SELECT   WorkOrder.OrderNo, WorkOrder.AutoOrderNo, WorkOrder.OrderDate, WorkOrder.CustomerID AS AccountName, '' AS Expr1, WOMFGDetails.WasteProduct, Products.BaseUOM, WorkOrder.Remarks, ISNULL(WOMFGDetails.WasteQty, 
                                 0) AS QtyIn, CASE WHEN SUM(WOMFGDetails.FinishedQuantity) > 0 THEN SUM(TotalFinishGoodCost) / SUM(WOMFGDetails.FinishedQuantity) ELSE 0 END AS InPrice, 0 AS QtyOut, 0 AS OutPrice, WorkOrder.Department, 
                                 WOMFGDetails.WastageStore, 'Work Order' AS DocumentType, 'IN' AS StockType, WorkOrder.BatchNo, WorkOrder.Location, WorkOrder.Company, WorkOrder.Project, WorkOrder.CreatedBy, WorkOrder.FiscalYear, 
                                 WorkOrder.IsApproved, Employee.EnableApprovals
                 FROM      WorkOrder INNER JOIN
                                 WOMFGDetails ON WorkOrder.OrderNo = WOMFGDetails.OrderNo INNER JOIN
                                 Employee ON WorkOrder.CreatedBy = Employee.Oid INNER JOIN
                                 Products ON WOMFGDetails.WasteProduct = Products.ProductID
                 WHERE   (WorkOrder.Status = 1) AND (WorkOrder.GCRecord IS NULL) AND (WorkOrder.IsVoided = 0)
                 GROUP BY WorkOrder.OrderNo, WorkOrder.AutoOrderNo, WorkOrder.OrderDate, WorkOrder.CustomerID, WOMFGDetails.WasteProduct, Products.BaseUOM, WorkOrder.Remarks, ISNULL(WOMFGDetails.WasteQty, 0), WorkOrder.Department, 
                                 WOMFGDetails.WastageStore, WorkOrder.BatchNo, WorkOrder.Location, WorkOrder.Company, WorkOrder.Project, WorkOrder.CreatedBy, WorkOrder.FiscalYear, WorkOrder.IsApproved, Employee.EnableApprovals
                 HAVING   (ISNULL(WOMFGDetails.WasteQty, 0) > 0)) AS T
WHERE   (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 WHEN EnableApprovals = 0 AND IsApproved = 1 THEN 1 ELSE 0 END) OR
                (IsApproved = CASE WHEN EnableApprovals = 1 THEN 1 ELSE 0 END)
UNION ALL
SELECT   OrderNo, AutoOrderNo, ReceivingDate, AccountName, Expr1, ProductID, UOMID, Remarks, QtyIn, InPrice, QtyOut, OutPrice, Department, StoreID, DocumentType, StockType, BatchNo, Location, Company, Project, CreatedBy, FiscalYear
FROM      (SELECT   WorkOrder_1.OrderNo, WorkOrder_1.AutoOrderNo, FinishedGoods.ReceivingDate, WorkOrder_1.CustomerID AS AccountName, '' AS Expr1, FinishedGoods.ProductID, FinishedGoods.UOMID, WorkOrder_1.Remarks, 
                                 FinishedGoods.ReceivingQty AS QtyIn, FinishedGoods.UnitCost AS InPrice, 0 AS QtyOut, 0 AS OutPrice, FinishedGoods.Department, FinishedGoods.StoreID, 'Work Order' AS DocumentType, 'IN' AS StockType, WorkOrder_1.BatchNo, 
                                 WorkOrder_1.Location, WorkOrder_1.Company, WorkOrder_1.Project, WorkOrder_1.Status, WorkOrder_1.CreatedBy, WorkOrder_1.FiscalYear, WorkOrder_1.IsApproved, Employee.EnableApprovals
                 FROM      WorkOrder AS WorkOrder_1 INNER JOIN
                                 Employee ON WorkOrder_1.CreatedBy = Employee.Oid INNER JOIN
                                 FinishedGoods ON WorkOrder_1.OrderNo = FinishedGoods.OrderNo
                 WHERE   (WorkOrder_1.GCRecord IS NULL) AND (FinishedGoods.IsVoided = 0) AND (WorkOrder_1.IsVoided = 0)) AS T
WHERE   (Status = 0) OR
                (Status = 1)) AS T INNER JOIN
                         Products ON T .ProductID = Products.ProductID ON ItemUnits.ID = T .UOMID INNER JOIN
                         Stores ON T .StoreID = Stores.StoreID
WHERE        (Products.IsActive = 1)) AS t1 INNER JOIN
UnitMaster ON t1.UOMID = UnitMaster.UOMID LEFT OUTER JOIN
ItemBatches ON t1.BatchNo = ItemBatches.BatchID WHERE PackQtyIn+PackQtyOut > 0 
