USE [eProc]
GO
/****** Object:  StoredProcedure [appauctionbid].[P_BidSubmission]    Script Date: 4/10/2024 7:24:07 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

/************************************************************************************
-- Purpose:Procedure use for Bid Submission , Bid Verification and Generate Extension.
-- Input Variables: @V_BidDetailStr - It is string containg insert query for TblAuctionBidDetail table.
-- Input Variables: @V_CheckBidAmtStr - This values supplied in case of Both type auction, here each item's governing column value should supplied by comma separated.
-- Output Variables: @V_OutputStr - returns output message as property file variables.
-- Author: Dipal
-- Creation Date: 15-03-2013 01:09PM

-- Modified By:Nirav Raval[Change Request #17957 Updation of Serial auction end date in tbl_Auction when extended]
-- Modify Date: 14-05-2014 04:00PM

-- Modified By:Nirav Raval[Update start price when it is 0 and first bid received in case of rank auction]
-- Modify Date: 02-06-2014 10:36AM

-- Modified By:Nirav Raval[Update start price when it is 0 and first bid received in case of Standard auction]
-- Modify Date: 13-06-2014 14:55 PM

-- Modified By:Nirav Raval[CR:19170 Provision of Proxy bid to be available for both Future and live events (Not applicable in serial auction)]
-- Modify Date: 19-07-2014 13:55 PM

-- Last Modified By: Saharsh Shah
-- Last Modification Date: 20 Oct, 2014

-- Last Modified By: Nirav Raval
-- Last Modification Date: 16 Jan, 2015 (Project Task #22517 - EMD Calculation in bidding process)

-- Last Modified By: Nirav Raval
-- Last Modification Date: 26 May, 2015 (Project Task #23847 - Need to give provision to have check on minimum quantity required)

-- Last Modified By: Tarun Pushp
-- Last Modification Date: 28 July, 2015 For optimisation added 'FAST_FORWARD' in cursor

-- Last Modified By: Nirav Raval
-- Last Modification Date: 31 July, 2015 For Change Request #25679 Need to give provision to consider <Increment/Decrement> amount with respect to <increment/decrement> in times 

-- Last Modified By: Nirav Raval
-- Last Modification Date: 09 Oct, 2015 For Bug #27538 - End date and time" displayed wrong on listing, when Extension is going on.

-- Last Modified By: Nirav Raval
-- Last Modification Date: 15 Oct, 2015 For Change Request  #27577 - Need to give provision to configure EMD for an event.

-- Last Modified By: Nirav Raval
-- Last Modification Date: 30 Oct, 2015 For Bug #27922 - Auction :: auto bid => system fire Extension when Bidder configure auto bid in last Extenstion minute

-- Last Modified By: Nirav Raval
-- Last Modification Date: 7 Dec, 2015 For CR #28406 - Auction :: Need to make bid capacity provision to be configuration based 

-- Last Modified By: Nirav Raval
-- Last Modification Date: 12 Dec, 2015 For BUG #28539,28540 - System should validate Bidder/Proxy Bidder with proper message if EMD payment is pending and Bidder tries to Submit a Bid

-- Last Modified By: Nirav Raval
-- Last Modification Date: 14 Dec, 2015 For BUG #28543 - Bidding Hall :: Available Bidding Capacity amount getting display wrong.

-- Last Modified By: Nirav Raval
-- Last Modification Date: 28 Jan, 2015 For CR #29283 - Bidding Hall :: Restrict bidder on H1
-- Last Modification Date: 29 April, 2016 For CR #27382 - Bidding Hall :: Allow minus for all number data type.

-- Last Modified By: Dhruvil Chokshi
-- Last Modification Date: 15 June, 2016 For Bug #31372 Auction :: System is not allowing bidder to bid if officer change auto extension mode during live event.

-- Last Modified By : Ashish Jani
-- Last Changes : Apply absolute function on incdecAmount in reverse negative value.

-- Last Modified By : Ashish Jani 
-- Last Changes : submit 0 bid enabled autobid. change l1h1amount to isFirstBid condition.
-- Last Modified By : Ashish Jani
-- Last Changes : apply all number changes in forwrd auction also.(#31891)

-- Last Modified By : Pradip Jinjala
-- Last Changes : 23 Nov, 2016 EMD wise Bid Capacity Configuration and Rest of Auction Money process.(#33318)

-- Last Modified By : Pradip Jinjala
-- Last Changes : 12 Dec, 2017 Auction ::Proxy bid :: System is not allowing to bid in future auction and same is allowing while auction is in live.(Bug #33853)

-- Last Modified By : Pradip Jinjala
-- Last Changes : 21 July, 2017 Project Task #38872 : Audit trial time of Bid submission should be same as shown in Bid History report. (English, Clock And Yankee Auction case)

-- Last Modified By : Pradip Jinjala
-- Last Changes : 27 Sep, 2017 Bug #43432 Auction :: System should fire proper extension when valid bid received in last minute


-- Last Modified By : Pradip Jinjala
-- Last Changes : 27 Dec, 2017 Project Task #50995 Auction :If L1 bidder is outbidded by other bidder, then the blocked EMD value shall be released and available amount shall be updated accordingly - This should be required in reverse auction also

-- Last Modified By : Pradip Jinjala
-- Last Changes : 24 May, 2018 Bug #61390: Auction::ICB::System should allow to bid with same amount if bidder's Bidding currency is Different.

-- Last Modified By : Indrajit Maheshwari
-- Last Changes : 04 Nov, 2022 - CR of Restrict bidder on H1/L1 and EMD - EMD Rollout functionality

-- Last Modified By : Indrajit Maheshwari
-- Last Changes : 20 Mar, 2023 - CR of Restrict by CAPTCHA Code during Auction Bidding for eGold SaaS

************************************************************************************/
ALTER PROCEDURE [appauctionbid].[P_BidSubmission] 
	@V_AuctionId INT,
	@V_BidderId INT,
	@V_TableId INT,
	@V_RowId INT,
	@V_BidPrice DECIMAL(20,5),
	@V_IPAddress VARCHAR(25),
	@V_BidType INT, -- 0 = Manual , 1 = Auto
	@V_SessionUserId INT,
	@V_UserDetailId INT,
	@V_BidDetailStr NVARCHAR(max),
	@V_CheckBidAmtStr NVARCHAR(max),
	@V_CalledFrom INT,
	@V_QtyBidPrice DECIMAL(20,5),
	@V_IsValidCaptcha INT OUTPUT  , -- 0 = FALSE, 1 = TRUE  
	@V_BidId INT OUTPUT	,
	@V_OutputStr VARCHAR(3000) OUTPUT,
	@V_bidSubmittedDate DATETIME OUTPUT
AS
BEGIN
	
	SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
	SET NOCOUNT ON;
	
	DECLARE @V_AuctionResult INT /* 1 = Grand Total, 2 = Item Wise, 3 = Both */,
			@V_AuctionType INT /* 1 = Standard, 2 = Rank  */,
			@V_EvenTypeId INT /* 1 = Forward, 2 = Reverse */,
			@V_BiddingType INT /* 1 = NCB , 2 = ICB */,
			@V_IncDecType INT /* 1 = Figure , 2 = Percentage */,
			@V_H1L1Amount DECIMAL(20,5) = 0.0 /* H1 or L1 amount */,
			@V_H1L1CurrencyId TINYINT = 0, 
			@V_IncDecInMultiple INT  /* 0 = No , 1 = Yes */,
			@V_IncDecAmount DECIMAL(20,5) /* Increment/Decrement Amount */,
			@V_StartPrice DECIMAL(20,5),
			@V_AuctionMode INT /* 1 = Open, 2 = Limited */,
			@V_IsBidderWiseStartPrice INT /* 0 = No, 1 = Yes */,
			@V_CheckReservePrice INT /* 0 = No , 1 = Yes */,
			@V_ReservePrice DECIMAL(20,5),
			@V_IsItemWiseTime INT  /* 0 = No , 1 = Yes */,
			@V_IsAutoExt INT /* 0 = No , 1 = Yes */,
			@V_AuctionStatus INT /*0 = Pending, 1 = Approved, 2 = Cancelled */,
			@V_FirstBidCond INT /* 1 = AuctinStartPrice , 2 = AuctionStartPrice+(-)Increment(Decrement)Value */,
			@V_AuctionEndDate DATETIME,
			@V_EndDateVirtual DATETIME,
			@V_StartDate DATETIME,
			@V_ExtMode INT  /* 1 = Fixed , 2 = Unlimited */,
			@V_ExtendBy INT  /* Extension Minute */,
			@V_ExtendWhen INT /* Extension accure if during this minute valid bid received */,
			@V_TotalExt INT ,
			@V_CurrentExt INT ,
			@V_BaseCurrencyId INT,
			@V_BidderCurrencyId INT ,
			@V_CheckBidPrice DECIMAL(20,5) = @V_BidPrice,
			@V_BidDate DATETIME,
			@V_IsValidBid INT = 1 /* 0 = No , 1 = Yes */,
			@V_Count INT = 0,
			@V_IsPvsH1Bidder INT = 0, 
			@V_RejectBidStatus INT = 0,
			@V_IsFirstBid INT = 0 /* 0 = Yes, 1 =  No */,
			@V_RankForExt INT = 0,
			@V_CStatus INT = 0,
			@V_DecimalValueUpto INT = 0,
			@V_QueryStr NVARCHAR(max) = '',
			@V_Temp varchar(35),
			@V_OldRank INT = 0,
			@V_IsValidAutoBid INT = 0 /* 0 = No , 1 = Yes */,
			@v_isAutoBidAllowed INT =0,		
			@v_configureTimeForItem INT = 0, /* 0 = Not set , 1 = Parallel, 2 = Serial */
			@V_SerialEndDateVirtual DATETIME,	/* Use for Serial Auction */
			/* Emd changes*/
			@V_IsEmdReq INT = 0,	/* 0 = No, 1 = Yes */
			@V_BiddingCapacity DECIMAL(20,5) = 0.0,	/* Use for Serial Auction */
			@V_EmdBalId INT = 0,
			@V_EmdBalance DECIMAL(20,5) = 0.0,
			@V_EmdAmount DECIMAL(20,5) = 0.0,
			@V_EmdUsed DECIMAL(20,5) = 0.0,			
			@V_bidderIdEmd INT = 0,
			@V_H2bidder INT = 0,
			@V_totalH1Bid DECIMAL(20,5) = 0.0,
			@V_ConfigEmdAmt DECIMAL(20,5) = 0.0,
			/* Start - For CR:23766 To have check on minimum quantity required */
			@V_isMinQtyChkReq INT = 0, 
			@V_minQtyBid DECIMAL(20,5) = 0.0,
			@V_maxQtyBid DECIMAL(20,5) = 0.0,
			@V_QtyColumnTypeId INT = 2,/* End - For CR:23766 To have check on minimum quantity required */
			@V_IsBidPriceIncDecInTimesReq INT = 0,
			@V_BidPriceIncDecInTimes INT = 0,
			@V_IsBidCapacityReq TINYINT = 0,
			@V_IsRestrictH1Bidder TINYINT = 0,
			@V_ItemH1Count INT = 0,
			@V_GovColDtType INT = 0,
			@V_IsRankExits INT = 0,
			@V_IsDisplayL1ItemWiseAndGTWise INT = 0,
			@V_TempCheckBidAmtStr VARCHAR(max) = @V_CheckBidAmtStr,
			@V_DelimIndex INT,
			@V_TempBidPrice DECIMAL(20,5) = 0.0,
			@V_TempRowId INT = 1,
			@V_IssecurityFees INT = 0,
			@V_emdPaymentMode INT = 0,
			@V_rankLogic INT = 0,
			@V_NoOfBidRestriction INT,
			@V_NoOfBidAllowed INT,
			@V_RemainBidAllowed INT = 0,
			
			/* Restrict Bidder on L1/H1 and EMD - 1: YES, 0: NO - EMD Rollout CR #9483 - Indrajit Maheshwari */
			@V_isRestrictBidderOnl1h1EMD INT = 0, 
			
			/* Auto Bid Notification CR #9679 - Indrajit Maheshwari */
			@V_AuctionBrief NVARCHAR(MAX) = '',
			@V_Subject VARCHAR(255) = '', 
			@V_Body NVARCHAR(MAX) = '', 
			@V_TableContents NVARCHAR(MAX) = '',
			@V_ItemName NVARCHAR(MAX) = '',
			@V_TableLimits NVARCHAR(MAX) = '',
			@V_AutoBidConfigOn DATETIME,
			@V_FromAddress VARCHAR(250) = '',
			@V_EmailId VARCHAR(250) = '', 
			@V_LinkId INT = 0, 
			@V_BidderIdAutoBid INT = 0,
			@V_LowerLimit NVARCHAR(MAX) = '',
			@V_UpperLimit NVARCHAR(MAX) = '',
			@V_Id INT = 1,
			/* byMessageBox Notification */
			@V_IsMsgBoxMailRequired INT = 0,
			@V_MsgBoxSubject NVARCHAR(MAX) = '',
			@V_MsgBoxBody NVARCHAR(500) = '',
			@V_ClientID INT = 0,

			@V_IsRestrictByCaptcha TINYINT = 0,
			@V_CaptchaRestrictEnableDurationInSec TINYINT = 0,
			@V_LastBidTimestamp DATETIME ;
			
    BEGIN TRANSACTION
    BEGIN TRY	
    
		set @V_IsValidCaptcha = 0;
		SET @V_BidId = 0 
		SET @V_OutputStr  = '' ;
		SELECT @V_BidDate = GETUTCDATE();
		print @V_BidDate;
		SELECT @V_GovColDtType = a.dataType FROM appauction.tbl_AuctionColumn a inner join appauction.tbl_AuctionGovColumn b on a.columnId = b.columnId and b.tableId=@V_TableId
		IF (@V_GovColDtType = 5 OR @V_BidPrice != 0.0 OR @V_BidType = 1)
		BEGIN
		
			/***Step: 1  Get Auction parameters ***/
			SELECT 
			@V_StartPrice = startPrice,@V_ReservePrice = reservePrice,@v_isAutoBidAllowed = isAutoBidAllowed,@V_ConfigEmdAmt = emdAmt	FROM appauction.tbl_AuctionCriteria WHERE auctionId = @V_AuctionId AND rowId = @V_RowId;		
			
			SELECT @V_IsPvsH1Bidder = count(*) FROM appauctionbid.tbl_AuctionBidRank WHERE auctionId = @V_AuctionId AND isActive = 1  AND bidderRank = 1 AND bidderId = @V_BidderId AND rowId = @V_RowId;			

			SELECT @V_BidDate = GETUTCDATE(),@V_AuctionResult = auctionResult,@V_BiddingType = biddingType,@V_EvenTypeId = eventTypeId,@V_AuctionType = auctionType, @V_IncDecType = incDecType,
				@v_configureTimeForItem = configureTimeForItem,@V_IncDecInMultiple = incDecInMultiple,@V_CheckReservePrice = checkReservePrice,@V_IsItemWiseTime = isItemWiseTime,@V_CStatus = cstatus,
				@V_RankForExt = rankForExt,@V_AuctionMode = auctionMode,@V_IsBidderWiseStartPrice = isBidderWiseStartPrice,@V_FirstBidCond = firstBidCond,@V_DecimalValueUpto = decimalValueUpto,
				@V_IsAutoExt = isAutoExt,@V_StartDate = startDate,@V_EndDateVirtual = endDateVirtual,@V_ExtMode = extMode,@V_ExtendBy = extendBy,@V_ExtendWhen  = extendWhen,@V_TotalExt = totalExt,@V_CurrentExt = currentExt,
				@V_IsEmdReq = isEmdReq, @V_BiddingCapacity = biddingCapacity, @V_isMinQtyChkReq = isMinQtyReq, @V_IsBidPriceIncDecInTimesReq = isBidPriceIncDecInTimesReq,
				@V_IsBidCapacityReq = isBidCapacityReq, @V_IsRestrictH1Bidder = isRestrictH1Bidder,@V_IsDisplayL1ItemWiseAndGTWise = isDisplayL1ItemWiseAndGTWise ,
				@V_IssecurityFees = securityFees,@V_emdPaymentMode = emdPaymentMode,@V_rankLogic=rankLogic,@V_NoOfBidRestriction = noOfBidRestriction,@V_NoOfBidAllowed = noOfBidAllowed 
				, @V_isRestrictBidderOnl1h1EMD = isRestrictBidderOnl1h1, @V_AuctionBrief = auctionBrief 
				FROM appauction.tbl_Auction	WHERE auctionId = @V_AuctionId;
			
			/* If Auctrion is Item wise, Standard, Limited and IsRestrictH1Bidder is Yes */
			IF(@V_AuctionResult = 2 AND @V_AuctionType = 1 AND @V_AuctionMode = 2 AND @V_IsRestrictH1Bidder = 1) 
			BEGIN
				/* Get Total count of items where this bidder is H1/L1 */
				SELECT @V_ItemH1Count = ISNULL(COUNT(1),0) FROM appauctionbid.tbl_AuctionBidRank WHERE auctionId = @V_AuctionId AND tableId = @V_TableId
				AND bidderId = @V_BidderId AND bidderRank = 1 AND isActive = 1
			END

			/* Start - For CR:23766 To have check on minimum quantity required */
			IF(@V_AuctionResult = 2 AND @V_isMinQtyChkReq = 1)/* If itemwise auction and if minimum qty check required is yes */
			BEGIN
				SELECT @V_minQtyBid = minQty FROM appauction.tbl_AuctionCriteria WHERE auctionId = @V_AuctionId AND rowId = @V_RowId
				
				SELECT @V_maxQtyBid = CONVERT(DECIMAL(20,5),tblauctioncell.cellValue) FROM appauction.tbl_AuctionCell tblauctioncell 
				INNER JOIN appauction.tbl_AuctionColumn tblauctioncolumn ON tblauctioncell.columnId = tblauctioncolumn.columnId
				WHERE tblauctioncolumn.tableId = @V_TableId AND tblauctioncell.rowId = @V_RowId AND tblauctioncolumn.columnTypeId = @V_QtyColumnTypeId								
			END
			/* END - For CR:23766 To have check on minimum quantity required */
			
			/* Start - For Change Request #25679 - Increment/Decrement In times logic */
			IF(@V_IncDecType = 1 AND @V_IsBidPriceIncDecInTimesReq = 1) /* If Inc/dec type in Figure and Bidding price increment/decrement in times required is yes */
			BEGIN
				SELECT @V_BidPriceIncDecInTimes = bidPriceIncDecInTimes FROM appauction.tbl_AuctionCriteria WHERE auctionId = @V_AuctionId AND rowId = @V_RowId
			END	
			/* End - For Change Request #25679 - Increment/Decrement In times logic */
			
			IF(@V_IsBidderWiseStartPrice = 1) /* Bidder wise start price set to yes */
			BEGIN
				SELECT @V_StartPrice = startPrice FROM appauction.tbl_AuctionBidderMap 
				WHERE auctionId = @V_AuctionId AND tableId = @V_TableId AND rowId = @V_RowId AND bidderId = @V_BidderId;
			END
			/*** Check Bidding Type = 'ICB' then convert bid amount into base currency  ***/
			IF(@V_BiddingType = 2)
			BEGIN
				/* Get bidder currency id */
				SELECT @V_BidderCurrencyId = B.currencyId 
				FROM appauctionbid.tbl_AuctionBidCurrency A
				INNER JOIN appauction.tbl_AuctionCurrency B ON A.auctionCurrencyId = B.auctionCurrencyId
				WHERE A.bidderId = @V_BidderId AND A.tableId = @V_TableId AND A.rowId = @V_RowId;
				
				/* Get auction Base currency  */
				SELECT @V_BaseCurrencyId = currencyId from appauction.tbl_AuctionCurrency WHERE auctionId = @V_AuctionId AND isDefault = 1;
			
				IF ( @V_BidderCurrencyId != @V_BaseCurrencyId)
				BEGIN 
					SELECT @V_StartPrice = appauction.F_CurrencyConvertion(@V_StartPrice, @V_BaseCurrencyId, @V_BidderCurrencyId, @V_BaseCurrencyId, @V_AuctionId, @V_DecimalValueUpto, @V_EvenTypeId, 0);
					SELECT @V_ReservePrice = appauction.F_CurrencyConvertion(@V_ReservePrice, @V_BaseCurrencyId, @V_BidderCurrencyId, @V_BaseCurrencyId, @V_AuctionId, @V_DecimalValueUpto, @V_EvenTypeId, 0);			
				END
			END
			
			/***Step: 2  Check auction is closed?, bidder is block listed? auction is Stop or Pause?  ***/
			SET  @V_Count = 1
			SELECT @V_Count=
				CASE WHEN (AM.configureTimeForItem = 2 AND AC.cstatus != 0) THEN
					CASE WHEN (@V_bidDate > AC.endDateVirtual) THEN 0 WHEN (@V_bidDate < AC.StartDate) THEN -1  ELSE 1	END 
				WHEN AM.configureTimeForItem = 2 AND AC.cstatus = 0 THEN 
					-1		
				WHEN AM.auctionResult = 2 AND AM.isItemWiseTime = 1 then			
					CASE WHEN (@V_bidDate > AC.endDateVirtual) THEN 0 WHEN (@V_bidDate < AC.StartDate) THEN -1  ELSE 1	END 
				ELSE
					CASE WHEN (@V_bidDate > AM.endDateVirtual) THEN 0 WHEN (@V_bidDate < AM.StartDate) THEN -1  ELSE 1	END 
				 END 
			 FROM appauction.tbl_Auction AM 
			 INNER JOIN appauction.tbl_AuctionCriteria AC ON AM.auctionId = AC.auctionId 
			 WHERE AM.auctionId=@V_AuctionId AND AC.rowId=@V_RowId
			 
			
					
			/* Modified By Nirav Raval For CR 19170 Allow Proxy Bid in Future Event*/ 
			IF(@V_BidderId = @V_SessionUserId AND @V_Count = -1) /* Check for bid started? */
			BEGIN
				SET @V_OutputStr = 'msg_auc_notstarted';
				SET @V_IsValidBid = 0;
				SET @V_RejectBidStatus  = 13;
			END
			ELSE 
			IF (@V_Count = 0 AND @V_CalledFrom != 1) /* Check for Bid time over? */
			BEGIN
				SET @V_OutputStr = 'msg_aucbid_bidtimeover';
				SET @V_IsValidBid = 0;
				SET @V_RejectBidStatus  = 5;
			END
			ELSE
			BEGIN
				 SET  @V_Count = 0
				 select @V_Count = COUNT(1) FROM appauctionbid.tbl_AuctionBidConfirmation WHERE auctionId = @V_AuctionId AND bidderId = @V_BidderId;
				 
				 IF(@V_Count < = 0 AND @V_BidType != 1) 
				 BEGIN
					SET @V_OutputStr = 'msg_auc_iagreeremain';
					SET @V_IsValidBid = 0;
					SET @V_RejectBidStatus  = 12;
				 END
				 ELSE
				 BEGIN
					 SET @V_AuctionStatus = 0;
					 SELECT TOP 1 @V_AuctionStatus = cstatus FROM appauction.tbl_AuctionStatus  WHERE auctionId = @V_AuctionId
					 AND (cstatus IN (2,3) OR (cstatus = 1 AND GETUTCDATE() BETWEEN stratdate AND endDate))
					 ORDER BY auctionStatusId DESC;
			 
					 IF (@V_CStatus = 2) /* Check for Auction Is cancelled? */
					 BEGIN
						SET @V_OutputStr = 'msg_auc_cancel';
						SET @V_IsValidBid = 0;
						SET @V_RejectBidStatus  = 11;
					 END
					 ELSE IF (@V_AuctionStatus = 1) /* Check for Auction Is paused? */
					 BEGIN
						SET @V_OutputStr = 'msg_auc_pause';
						SET @V_IsValidBid = 0;
						SET @V_RejectBidStatus  = 7;
					 END
					 ELSE IF (@V_AuctionStatus = 2) /* Check for Auction Is stoped? */
					 BEGIN
						SET @V_OutputStr = 'msg_auc_stop';
						SET @V_IsValidBid = 0;
						SET @V_RejectBidStatus  = 8;
					 END
					
					 ELSE
					 BEGIN 				 
						 SET  @V_Count = 0;
						 SELECT @V_Count = COUNT(1) FROM appauctionbid.tbl_AuctionBlockBidder 
						 WHERE auctionId = @V_AuctionId AND bidderid = @V_BidderId AND isBlocked = 1;
						 
						 IF (@V_Count > 0) /* Check for Bidder was blocked? */
						 BEGIN
							SET @V_OutputStr = 'msg_aucbid_bidderblock';
							SET @V_IsValidBid = 0;
							SET @V_RejectBidStatus  = 6;
						 END
						 ELSE IF(@V_NoOfBidRestriction = 1)
						 BEGIN 
							IF(@V_AuctionResult = 2)
							BEGIN
								select @V_NoOfBidAllowed = noOfBidAllowed from appauction.tbl_AuctionCriteria where auctionId = @V_AuctionId and rowId = @V_RowId
								select @V_RemainBidAllowed = (@V_NoOfBidAllowed-ISNULL(Count(bidId),0)) from appauctionbid.tbl_auctionbid where auctionId = @V_AuctionId and rowId = @V_RowId and bidderId = @V_BidderId and isApproved = 1 and submittedBy = bidderId
							END
							ELSE 
							BEGIN 
								select @V_RemainBidAllowed = (@V_NoOfBidAllowed-ISNULL(Count(bidId),0)) from appauctionbid.tbl_auctionbid where auctionId = @V_AuctionId and bidderId = @V_BidderId and isApproved = 1 and submittedBy = bidderId
							END
							IF (@V_RemainBidAllowed = 0 and @V_BidderId = @V_SessionUserId) /* Check bid is Remaining for bidder or not? */
							BEGIN
								SET @V_OutputStr = 'msg_aucbid_no_remainBid';
								SET @V_IsValidBid = 0;
								SET @V_RejectBidStatus  = 25;
							END
						 END
						 /* Start - For CR:23766 To have check on minimum quantity required */
						 ELSE IF (@V_AuctionResult = 2 AND @V_isMinQtyChkReq = 1)
						 BEGIN
							IF (@V_QtyBidPrice < @V_minQtyBid)
							BEGIN
								SET @V_OutputStr = 'msg_auc_bid_minqtycheck';
								SET @V_IsValidBid = 0;
								SET @V_RejectBidStatus  = 15;
							END
							ELSE IF (@V_QtyBidPrice > @V_maxQtyBid)
							BEGIN
								SET @V_OutputStr = 'msg_auc_bid_maxqtycheck';
								SET @V_IsValidBid = 0;
								SET @V_RejectBidStatus  = 16;
							END
						 END
						 /* End - For CR:23766 To have check on minimum quantity required */
					 END	
				 END
			 END
			 IF(@V_RejectBidStatus = 0 AND @V_BidType = 1) /* Auto Bid */
			 BEGIN
				exec appauctionbid.p_AutoBid @v_AuctionId,@V_TableId,@V_RowId,@V_IPAddress,@V_SessionUserId,@V_UserDetailId,@V_BidDate,@V_CalledFrom,@V_IsValidBid output;								
			 END
			 ELSE
			 BEGIN
				 /**** Check in case of auction type is both(Grandtotal+Itemwise) *******/
				 IF(@V_AuctionResult = 3 AND @V_RejectBidStatus = 0)
				 BEGIN
					SET @V_RejectBidStatus = 0;
					exec appauctionbid.p_ValidateGrandTotalBid @V_auctionId  ,@V_BidderId  ,@V_TableId,@V_SessionUserId ,@V_CheckBidAmtStr ,@V_RejectBidStatus output;
					IF(@V_RejectBidStatus ! =  0)
					BEGIN
						SET @V_OutputStr = 'msg_aucbid_invalidbid';
						SET @V_IsValidBid = 0;
					END
				 END						 
				 
				 /***Step: 3  Get H1L1Amount  ***/			
					IF (@V_AuctionType = 1) /* Standard Auction */
					BEGIN
						SELECT @V_IsFirstBid = COUNT(1) FROM appauctionbid.tbl_AuctionBidRank
						WHERE auctionId = @V_AuctionId AND tableId = @V_TableId AND rowId = @V_RowId AND isActive = 1;
						
						IF (@V_EvenTypeId = 1 AND @V_IsEmdReq = 1 AND (@V_IsBidCapacityReq = 1 OR @V_IsBidCapacityReq = 2)) /* If forward auction and emd is enabled with Bid capacity required as Yes*/
						BEGIN
							/* Get EMD Balance amount which will utilised against each successful bid */
							SELECT  @V_EmdBalance = emdBalance, @V_EmdBalId = emdId FROM appauction.tbl_EmdBalance WHERE auctionId = @V_AuctionId AND bidderId = @V_BidderId							
						END
						 
						IF (@V_EvenTypeId = 2 AND @V_IsEmdReq = 1 AND @V_IsBidCapacityReq = 2 AND @V_IssecurityFees = 2 AND @V_AuctionResult = 2 AND @V_emdPaymentMode = 1) /* If reverse auction and emd is enabled with Bid capacity required as Emd wise*/
						BEGIN
							/* Get EMD Balance amount which will utilised against each successful bid */
							SELECT  @V_EmdBalance = emdBalance, @V_EmdBalId = emdId FROM appauction.tbl_EmdBalance WHERE auctionId = @V_AuctionId AND bidderId = @V_BidderId							
						END
						
						/* Indrajit Maheshwari - EMD Rollout - EMD Balance Check - START */
						IF(@V_isRestrictBidderOnl1h1EMD = 1 AND @V_AuctionResult = 2 AND @V_emdPaymentMode = 1 AND @V_IsEmdReq = 2)
						BEGIN
							SELECT  @V_EmdBalance = emdBalance, @V_EmdBalId = emdId FROM appauction.tbl_EmdBalance WHERE auctionId = @V_AuctionId AND bidderId = @V_BidderId
						END
						/* Indrajit Maheshwari - EMD Rollout - EMD Balance Check - END */

						IF (@V_BiddingType =  2)
						BEGIN 
							SELECT @V_H1L1Amount = (CASE WHEN ABCD.currencyId = @V_BidderCurrencyId THEN AB.bidderBidPrice ELSE ABR.bidPrice END), @V_H1L1CurrencyId = ABCD.currencyId  
							FROM appauctionbid.tbl_AuctionBidRank ABR 
							INNER JOIN appauctionbid.tbl_AuctionBid AB  ON ABR.bidId = AB.bidId 
							INNER JOIN appauctionbid.tbl_AuctionBidCurrency ABCU ON ABCU.tableId = ABR.tableId AND  ABCU.rowId = ABR.rowId AND ABCU.bidderId = ABR.bidderId
							INNER JOIN appauction.tbl_AuctionCurrency ABCD on ABCD.auctionCurrencyId = ABCU.auctionCurrencyId 
							WHERE ABR.auctionId = @V_AuctionId AND ABR.tableId = @V_TableId AND ABR.rowId = @V_RowId AND ABR.isActive = 1 AND ABR.bidderRank = 1;
						END	
						ELSE 
						
						
						BEGIN 
							SELECT @V_H1L1Amount = ABR.bidPrice 
							FROM appauctionbid.tbl_AuctionBidRank ABR 							
							WHERE ABR.auctionId = @V_AuctionId AND ABR.tableId = @V_TableId AND ABR.rowId = @V_RowId AND ABR.isActive = 1 AND ABR.bidderRank = 1;
						END 	
						
						IF(@V_H1L1Amount IS NULL)
						BEGIN
							SET @V_H1L1Amount = 0.0;
						END
					END
					ELSE IF(@V_AuctionType = 2 AND @V_EvenTypeId = 1) /* Rank Auction and Forward Auction */
					BEGIN
						SELECT @V_IsFirstBid = COUNT(1) FROM appauctionbid.tbl_AuctionBidRank
						WHERE auctionId = @V_AuctionId AND tableId = @V_TableId AND rowId = @V_RowId AND bidderId = @V_BidderId AND isActive = 1;
					
						IF (@V_EvenTypeId = 1 AND @V_IsEmdReq = 1 AND @V_IsBidCapacityReq = 1) /* If forward auction and emd is enabled with Bid capacity required as Yes*/
						BEGIN
							/* Get EMD Balance amount which will utilised against each successful bid */
							SELECT  @V_EmdBalance = emdBalance, @V_EmdBalId = emdId FROM appauction.tbl_EmdBalance WHERE auctionId = @V_AuctionId AND bidderId = @V_BidderId							
						END

						/* Indrajit Maheshwari - EMD Rollout - EMD Balance Check - START */
						IF(@V_isRestrictBidderOnl1h1EMD = 1 AND @V_AuctionResult = 2 AND @V_emdPaymentMode = 1 AND @V_IsEmdReq = 2)
						BEGIN
							SELECT  @V_EmdBalance = emdBalance, @V_EmdBalId = emdId FROM appauction.tbl_EmdBalance WHERE auctionId = @V_AuctionId AND bidderId = @V_BidderId
						END
						/* Indrajit Maheshwari - EMD Rollout - EMD Balance Check - END */
						
						IF(@V_rankLogic = 0)
						BEGIN 
							IF(@V_BiddingType = 2)
							BEGIN 
								SET @V_H1L1CurrencyId = @V_BidderCurrencyId;
								SELECT @V_H1L1Amount = ISNULL(MAX(bidderBidPrice),0) FROM appauctionbid.tbl_AuctionBid 
								WHERE isApproved = 1 AND bidderId = @V_BidderId AND auctionId = @V_AuctionId AND tableId = @V_TableId AND rowId = @V_RowId;
							END						 
							ELSE
							BEGIN
								SELECT @V_H1L1Amount = ISNULL(MAX(bidPrice),0) FROM appauctionbid.tbl_AuctionBid 
								WHERE isApproved = 1 AND bidderId = @V_BidderId AND auctionId = @V_AuctionId AND tableId = @V_TableId AND rowId = @V_RowId;
							END
						END
						ELSE
						BEGIN
							SET  @V_Count = 0;
							SELECT @V_Count = COUNT(bidPrice) FROM appauctionbid.tbl_AuctionBid 
							WHERE isApproved = 1 AND bidPrice = @V_BidPrice AND auctionId = @V_AuctionId AND tableId = @V_TableId AND rowId = @V_RowId AND bidderId != @V_BidderId ;
							IF(@V_BiddingType = 2)
							BEGIN 
								SET @V_H1L1CurrencyId = @V_BidderCurrencyId;
								
								IF(@V_Count = 0)
								BEGIN
									SELECT @V_H1L1Amount = ISNULL(MAX(bidderBidPrice),0) FROM appauctionbid.tbl_AuctionBid 
									WHERE isApproved = 1 AND bidderId = @V_BidderId AND auctionId = @V_AuctionId AND tableId = @V_TableId AND rowId = @V_RowId;
								END
								ELSE
								BEGIN
									SET @V_OutputStr = 'msg_samebid_not_accepted';
									SET @V_IsValidBid = 0;
									SET @V_RejectBidStatus  = 23;
								END
							END						 
							ELSE
							BEGIN
								IF(@V_Count = 0)
								BEGIN
									SELECT @V_H1L1Amount = ISNULL(MAX(bidPrice),0) FROM appauctionbid.tbl_AuctionBid 
									WHERE isApproved = 1 AND bidderId = @V_BidderId AND auctionId = @V_AuctionId AND tableId = @V_TableId AND rowId = @V_RowId;
								END
								ELSE
								BEGIN
									SET @V_OutputStr = 'msg_samebid_not_accepted';
									SET @V_IsValidBid = 0;
									SET @V_RejectBidStatus  = 23;
								END
							END
						END
					END
					ELSE IF(@V_AuctionType = 2 AND @V_EvenTypeId = 2) /* Rank Auction and Reverse Auction */
					BEGIN
						SELECT @V_IsFirstBid = COUNT(1) FROM appauctionbid.tbl_AuctionBidRank
						WHERE auctionId = @V_AuctionId AND tableId = @V_TableId AND rowId = @V_RowId AND bidderId = @V_BidderId AND isActive = 1;
						IF(@V_rankLogic = 0)
						BEGIN 
							IF(@V_BiddingType = 2)
							BEGIN 
								SET @V_H1L1CurrencyId = @V_BidderCurrencyId;
								SELECT @V_H1L1Amount = ISNULL(MIN(bidderBidPrice),0) FROM appauctionbid.tbl_AuctionBid 
								WHERE isApproved = 1 AND bidderId = @V_BidderId AND auctionId = @V_AuctionId AND tableId = @V_TableId AND rowId = @V_RowId;
							END
							ELSE
							BEGIN
								SELECT @V_H1L1Amount = ISNULL(MIN(bidPrice),0) FROM appauctionbid.tbl_AuctionBid 
								WHERE isApproved = 1 AND bidderId = @V_BidderId AND auctionId = @V_AuctionId AND tableId = @V_TableId AND rowId = @V_RowId;
							END
						END
						ELSE
						BEGIN
							SET  @V_Count = 0;
							
							SELECT @V_Count = COUNT(bidPrice) FROM appauctionbid.tbl_AuctionBid 
							WHERE isApproved = 1 AND bidderBidPrice = @V_BidPrice AND auctionId = @V_AuctionId AND tableId = @V_TableId AND rowId = @V_RowId AND bidderId != @V_BidderId ;

							IF(@V_BiddingType = 2)
							BEGIN 
								SET @V_H1L1CurrencyId = @V_BidderCurrencyId;

								IF(@V_Count = 0)
								BEGIN
									SELECT @V_H1L1Amount = ISNULL(MIN(bidderBidPrice),0) FROM appauctionbid.tbl_AuctionBid 
									WHERE isApproved = 1 AND bidderId = @V_BidderId AND auctionId = @V_AuctionId AND tableId = @V_TableId AND rowId = @V_RowId;
								END
								ELSE
								BEGIN
									SET @V_OutputStr = 'msg_samebid_not_accepted';
									SET @V_IsValidBid = 0;
									SET @V_RejectBidStatus  = 23;
								END
							END
							ELSE
							BEGIN
								IF(@V_Count = 0)
								BEGIN
									SELECT @V_H1L1Amount = ISNULL(MIN(bidPrice),0) FROM appauctionbid.tbl_AuctionBid 
									WHERE isApproved = 1 AND bidderId = @V_BidderId AND auctionId = @V_AuctionId AND tableId = @V_TableId AND rowId = @V_RowId;
								END
								ELSE
								BEGIN
									SET @V_OutputStr = 'msg_samebid_not_accepted';
									SET @V_IsValidBid = 0;
									SET @V_RejectBidStatus  = 23;
								END
							END
						END
						
						/* Indrajit Maheshwari - EMD Rollout - EMD Balance Check - START */
						IF(@V_isRestrictBidderOnl1h1EMD = 1 AND @V_AuctionResult = 2 AND @V_emdPaymentMode = 1 AND @V_IsEmdReq = 2)
						BEGIN
							SELECT  @V_EmdBalance = emdBalance, @V_EmdBalId = emdId FROM appauction.tbl_EmdBalance WHERE auctionId = @V_AuctionId AND bidderId = @V_BidderId
						END
						/* Indrajit Maheshwari - EMD Rollout - EMD Balance Check - END */
					END
					
					IF (@V_BiddingType = 2 AND @V_BidderCurrencyId != @V_BaseCurrencyId AND @V_BidderCurrencyId != @V_H1L1CurrencyId)
					BEGIN 
						SELECT @V_H1L1Amount = appauction.F_CurrencyConvertion(@V_H1L1Amount, @V_BaseCurrencyId, @V_BidderCurrencyId, @V_BaseCurrencyId, @V_AuctionId, @V_DecimalValueUpto, @V_EvenTypeId, 0);						
					END		
									
					IF (@V_IsFirstBid != 0 AND @V_AuctionType = 1 AND @V_IsBidderWiseStartPrice = 1 AND ((@V_EvenTypeId = 1 AND @V_StartPrice > @V_H1L1Amount) OR (@V_EvenTypeId = 2 AND @V_StartPrice < @V_H1L1Amount))) /* Standard Auction and bidder wise start price is yes */
					BEGIN
						SELECT @V_IsFirstBid = COUNT(1) FROM appauctionbid.tbl_AuctionBidRank
						WHERE auctionId = @V_AuctionId AND tableId = @V_TableId AND rowId = @V_RowId AND bidderId = @V_BidderId AND isActive = 1;
						IF(@V_IsFirstBid = 0) /* If no bid received from that bidder */
						BEGIN
							SET @V_H1L1Amount = 0.0;
						END				
					END		
					
				 
				 
				 /**** Check wether autobid is enabled? *******/
				 SET  @V_Count = 0;
				 SELECT @V_Count = ISNULL(bidderRank,0) FROM appauctionbid.tbl_AuctionBidRank
				 WHERE auctionId=@V_AuctionId and rowId=@V_RowId and bidderId=@V_BidderId  AND isActive = 1;
				 print @V_Count;
				 IF(@V_RejectBidStatus = 0 AND @v_isAutoBidAllowed = 1 AND @V_Count <=1) /* Auto bid allowed */
				 BEGIN
					 SET  @V_Count = 0
					 IF(@V_EvenTypeId = 1) /* Forward Auction */
					 BEGIN
						SELECT  @V_Count = COUNT(1) FROM appauctionbid.tbl_AutoBid
						WHERE auctionId=@V_AuctionId AND (@V_IsFirstBid = 0 OR bestPrice > @V_H1L1Amount)
						AND isActive=1 AND rowId=@V_RowId AND bidderId=@V_BidderId						
					 END  
					 ELSE --IF (@V_EvenTypeId = 2) /* Reverse Auction  */
					 BEGIN
						SELECT  @V_Count = COUNT(1) FROM appauctionbid.tbl_AutoBid
						WHERE auctionId=@V_AuctionId AND (@V_IsFirstBid = 0 OR bestPrice < @V_H1L1Amount)
						AND isActive=1 AND rowId=@V_RowId AND bidderId=@V_BidderId 
					 END	
					 
					 IF(@V_Count > 0)
					 BEGIN
						SET @V_OutputStr = 'msg_aucbid_autobidenabled';
						SET @V_IsValidBid = 0;
						SET @V_RejectBidStatus  = 14;
					 END
				 END
				 
				
				 IF(@V_RejectBidStatus = 0)
			 	 BEGIN 
					SET @V_CheckBidPrice = @V_BidPrice;
					/***Step: 4  Reserve Price Check ***/
					
					IF(@V_CheckReservePrice = 1 AND @V_EvenTypeId = 1 AND @V_CheckBidPrice > @V_ReservePrice)  /* Reserve price check = yes and Forward auction */
					BEGIN
						SET @V_OutputStr = 'msg_aucbid_higherreserveprice';
						SET @V_IsValidBid = 0;
						SET @V_RejectBidStatus  = 1;
					END
					ELSE IF(@V_CheckReservePrice = 1 AND @V_EvenTypeId = 2 AND @V_CheckBidPrice < @V_ReservePrice)  /* Reserve price check = yes and Reverse auction */
					BEGIN
						SET @V_OutputStr = 'msg_aucbid_lowerreserveprice';
						SET @V_IsValidBid = 0;
						SET @V_RejectBidStatus  = 1;
					END
					
					/* PRINT('REserver price check End '+convert(varchar,@V_IsValidBid)) */
					/***Step: 5  Get Decrement/Increment by identifing auction running under which duration in extension or orginal period? Validate bid amount base on that ***/
					IF(@V_IsValidBid ! =  0)
					BEGIN
						IF (@V_IsFirstBid = 0 AND @V_StartPrice = 0) /* If auction first bid and start price is 0 */
						BEGIN
							SET @V_H1L1Amount = @V_CheckBidPrice;
							SET @V_StartPrice = @V_CheckBidPrice;
							SET @V_IsValidBid = 1;
							SET @V_OutputStr = 'msg_aucbid_success';
							IF (@V_IsValidBid = 1 AND ((@V_EvenTypeId = 1 AND @V_IsEmdReq = 1 AND (@V_IsBidCapacityReq = 1 OR @V_IsBidCapacityReq = 2)) OR (@V_EvenTypeId = 2 AND @V_IsEmdReq = 1 AND @V_IsBidCapacityReq = 2 AND @V_IssecurityFees = 2 AND @V_AuctionResult = 2 AND @V_emdPaymentMode = 1))) /* If emd is enabled and Bid Capacity required as Yes or EMD wise and Auction is Forward or Reverse*/
							BEGIN		
								IF(@V_IsBidCapacityReq = 1)
								BEGIN
									IF ((@V_CheckBidPrice / @V_BiddingCapacity) > @V_EmdBalance)
									BEGIN
										/* Reject bid because of EMD balance is sufficient with respect to bid */
										SET @V_OutputStr = 'msg_aucbid_emdinsufficient';
										SET @V_IsValidBid = 0;
										SET @V_RejectBidStatus  = 21;
									END
								END
								/* Reject bid because of EMD balance is sufficient with respect to bid FOR V_IsBidCapacityReq == 2*/
								ELSE IF(@V_IsBidCapacityReq = 2) 
								BEGIN
									IF(@V_EmdBalance < @V_ConfigEmdAmt)
									BEGIN
										SET @V_OutputStr = 'msg_auc_payemd';
										SET @V_IsValidBid = 0;
										SET @V_RejectBidStatus  = 21;
									END
								END
							END

							/* Indrajit Maheshwari - EMD Rollout - Insufficient EMD Balance Check */
							IF(@V_IsValidBid = 1 AND @V_IsEmdReq = 2 AND @V_isRestrictBidderOnl1h1EMD = 1 AND @V_AuctionResult = 2 AND @V_emdPaymentMode = 1)
							BEGIN
								IF(@V_EmdBalance < @V_ConfigEmdAmt AND @V_IsPvsH1Bidder != 1)
									BEGIN
										SET @V_OutputStr = 'msg_aucbid_insufficient_emd_balance';
										SET @V_IsValidBid = 0;
										SET @V_RejectBidStatus  = 28;
								END
							END
							/* EMD Rollout - Insufficient EMD Balance Check END*/

							/* If Auctrion is Item wise, Standard, Limited, IsRestrictH1Bidder is Yes and Bidder is Rank 1 in 1 Item*/
							IF(@V_AuctionResult = 2 AND @V_AuctionType = 1 AND @V_IsRestrictH1Bidder = 1 AND @V_AuctionMode = 2 AND @V_ItemH1Count = 1) 
							BEGIN
								SET @V_OutputStr = 'msg_auction_restrictH1Bidder';
								SET @V_IsValidBid = 0;
								SET @V_RejectBidStatus  = 18;
							END

							IF(@V_AuctionType = 1 AND @V_IsValidBid = 1) /* If Standarad Auction */
							BEGIN
								UPDATE appauction.tbl_AuctionCriteria
								SET startPrice = (CASE WHEN @V_BiddingType = 2 AND @V_BidderCurrencyId != @V_BaseCurrencyId 
														THEN appauction.F_CurrencyConvertion(@V_StartPrice, @V_BidderCurrencyId, @V_BaseCurrencyId, @V_BaseCurrencyId, @V_AuctionId, @V_DecimalValueUpto, @V_EvenTypeId, 0)
													  ELSE @V_StartPrice END)
								WHERE auctionId = @V_AuctionId AND rowId = @V_RowId;
							END	
						END /* END of (@V_IsFirstBid = 0 AND @V_StartPrice = 0)  */
						ELSE /* Start price is not 0 or is not first bid */
						BEGIN		
							IF(@V_IsItemWiseTime = 1) /* If Item wise time required = Yes  */
							BEGIN
								SELECT @V_IncDecAmount = 
								CASE WHEN (B.isAutoExt = 1  AND B.endDate ! =  B.endDateVirtual AND @V_BidDate < =  B.endDateVirtual) THEN
									B.incDecOnExt	
								WHEN (A.isIncDecInPeriod = 1  AND @V_BidDate BETWEEN B.timeForIncDecToItem AND B.endDate) THEN
									B.incDecValOnTime
								WHEN (@V_BidDate BETWEEN B.startDate AND B.endDateVirtual) THEN
									B.incDecValue
								WHEN (@V_BidDate <= A.startDate AND @V_BidderId != @V_SessionUserId) THEN
									B.incDecValue
  								END
								FROM appauction.tbl_Auction A
								INNER JOIN appauction.tbl_AuctionCriteria B ON A.auctionId = B.auctionId 
								WHERE A.auctionId = @V_AuctionId AND B.rowId = @V_RowId ;
							END
							ELSE  /* Auction result is "Grand total wise/both" or Itemwise time = No */
							BEGIN
								SELECT @V_IncDecAmount = 
								CASE WHEN (A.isAutoExt = 1  AND A.endDate ! =  A.endDateVirtual AND @V_BidDate < =  A.endDateVirtual) THEN
									B.incDecOnExt	
								WHEN (A.isIncDecInPeriod = 1  AND @V_BidDate BETWEEN A.timeForIncDecToItem AND A.endDate) THEN
									B.incDecValOnTime
								WHEN (@V_BidDate BETWEEN A.startDate AND A.endDateVirtual) THEN
									B.incDecValue
								WHEN (@V_BidDate <= A.startDate AND @V_BidderId != @V_SessionUserId) THEN
									B.incDecValue
  								END
								FROM appauction.tbl_Auction A
								INNER JOIN appauction.tbl_AuctionCriteria B ON A.auctionId = B.auctionId 
								WHERE A.auctionId = @V_AuctionId AND B.rowId = @V_RowId ;							
							END
														
							IF (@V_BiddingType = 2 AND @V_BidderCurrencyId != @V_BaseCurrencyId)
							BEGIN 
								SELECT @V_IncDecAmount = appauction.F_CurrencyConvertion(@V_IncDecAmount, @V_BaseCurrencyId, @V_BidderCurrencyId, @V_BaseCurrencyId, @V_AuctionId, @V_DecimalValueUpto, @V_EvenTypeId, 1);
							END
							/* PRINT('@V_CheckBidPrice ='+convert(varchar,@V_CheckBidPrice)+'Inc/Dec amount = '+convert(varchar,@V_IncDecAmount)+'H1L1amount = '+convert(varchar,@V_H1L1Amount)+'@V_StartPrice = '+convert(varchar,@V_StartPrice))  */					
							IF (@V_IsFirstBid = 0 AND @V_H1L1Amount = 0 AND @V_StartPrice ! =  0) -- If first bid and Auction Start price is not 0?
							BEGIN
								SET @V_H1L1Amount = @V_StartPrice;
							END
							
							IF (@V_IncDecType = 2)	 /* Increment /Decrement in percentage? */
							BEGIN
								SET @V_IncDecAmount = (@V_H1L1Amount * @V_IncDecAmount)/100;
							END
							
							IF(@V_EvenTypeId = 1) /* Forward Auction */
							BEGIN		
								/* Start - For Change Request #25679 - Increment/Decrement In times logic */						
								IF(@V_IncDecType = 1 AND @V_IsBidPriceIncDecInTimesReq = 1 AND (@V_CheckBidPrice > (@V_H1L1Amount + (ABS(@V_IncDecAmount) * @V_BidPriceIncDecInTimes))))
								BEGIN									
									SET @V_OutputStr = 'msg_aucbid_incdecintimes';
									SET @V_IsValidBid = 0;
									SET @V_RejectBidStatus  = 17;
								END /* End - For Change Request #25679 - Increment/Decrement In times logic */
								ELSE IF (((@V_StartPrice !=  0 AND @V_AuctionType = 2) OR (@V_AuctionType = 1)) AND @V_CheckBidPrice < @V_StartPrice) /* First bid condition = Auction Start Price and bid received less than start price? */
								BEGIN
									SET @V_OutputStr = 'msg_aucbid_incrementstartpricecriteria';
									SET @V_IsValidBid = 0;
									SET @V_RejectBidStatus  = 9;
								END
								ELSE IF (@V_IsFirstBid = 0 AND @V_FirstBidCond = 1 AND @V_CheckBidPrice >=  @V_StartPrice) /* First bid condition = Auction Start Price? */
								BEGIN
									SET @V_IsValidBid = 1;
								END
								ELSE IF (@V_CheckBidPrice >=  (@V_H1L1Amount + ABS(@V_IncDecAmount)))
								BEGIN
									SET @V_IsValidBid = 1;
									IF(@V_BidDate <= @V_StartDate AND @V_BidderId != @V_SessionUserId)
									BEGIN
										SELECT @V_Count = COUNT(1) FROM appauctionbid.tbl_AuctionBidRank
										WHERE auctionId = @V_AuctionId AND tableId = @V_TableId AND rowId = @V_RowId AND bidderId = @V_BidderId AND isActive = 1;
										IF(@V_Count != 0)
										BEGIN
											SET @V_IsValidBid = 0;
										END
									END
								END
								ELSE
								BEGIN
									SET @V_OutputStr = 'msg_aucbid_incrementcriteria';
									SET @V_IsValidBid = 0;
									SET @V_RejectBidStatus  = 2;
								END
								
								/* Start - For CR:23766 To have check on minimum quantity required */
								 IF (@V_AuctionResult = 2 AND @V_isMinQtyChkReq = 1)
								 BEGIN
									IF (@V_QtyBidPrice < @V_minQtyBid)
									BEGIN
										SET @V_OutputStr = 'msg_auc_bid_minqtycheck';
										SET @V_IsValidBid = 0;
										SET @V_RejectBidStatus  = 15;
									END
									ELSE IF (@V_QtyBidPrice > @V_maxQtyBid)
									BEGIN
										SET @V_OutputStr = 'msg_auc_bid_maxqtycheck';
										SET @V_IsValidBid = 0;
										SET @V_RejectBidStatus  = 16;
									END
								 END
								 /* End - For CR:23766 To have check on minimum quantity required */

								/* Add here emd validation check */
								IF (@V_IsValidBid = 1 AND ((@V_EvenTypeId = 1 AND @V_IsEmdReq = 1 AND (@V_IsBidCapacityReq = 1 OR @V_IsBidCapacityReq = 2)) OR (@V_EvenTypeId = 2 AND @V_IsEmdReq = 1 AND @V_IsBidCapacityReq = 2 AND @V_IssecurityFees = 2 AND @V_AuctionResult = 2 AND @V_emdPaymentMode = 1))) /* If emd is enabled and Bid Capacity required as Yes or EMD wise and Auction is Forward or Reverse*/
								BEGIN
									IF(@V_IsBidCapacityReq = 1)
									BEGIN
										IF ((@V_CheckBidPrice / @V_BiddingCapacity) > @V_EmdBalance)
										BEGIN
											/* Reject bid because of EMD balance is sufficient with respect to bid */
											SET @V_OutputStr = 'msg_aucbid_emdinsufficient';
											SET @V_IsValidBid = 0;
											SET @V_RejectBidStatus  = 21;
										END
									END
									/* Reject bid because of EMD balance is sufficient with respect to bid FOR V_IsBidCapacityReq == 2*/
									ELSE IF(@V_IsBidCapacityReq = 2) 
									BEGIN
										IF(@V_EmdBalance < @V_ConfigEmdAmt)
										BEGIN
											SET @V_OutputStr = 'msg_auc_payemd';
											SET @V_IsValidBid = 0;
											SET @V_RejectBidStatus  = 21;
										END
									END
								END

								/* Indrajit Maheshwari - EMD Rollout - Insufficient EMD Balance Check */
								IF(@V_IsValidBid = 1 AND @V_IsEmdReq = 2 AND @V_isRestrictBidderOnl1h1EMD = 1 AND @V_AuctionResult = 2 AND @V_emdPaymentMode = 1)
								BEGIN
									IF(@V_EmdBalance < @V_ConfigEmdAmt AND @V_IsPvsH1Bidder != 1)
										BEGIN
											SET @V_OutputStr = 'msg_aucbid_insufficient_emd_balance';
											SET @V_IsValidBid = 0;
											SET @V_RejectBidStatus  = 28;
									END
								END
								/* EMD Rollout - Insufficient EMD Balance Check END*/
								
								/* If Auctrion is Item wise, Standard, Limited, IsRestrictH1Bidder is Yes and Bidder is Rank 1 in 1 Item*/
								IF(@V_AuctionResult = 2 AND @V_AuctionType = 1 AND @V_IsRestrictH1Bidder = 1 AND @V_AuctionMode = 2 AND @V_ItemH1Count = 1) 
								BEGIN
									SET @V_OutputStr = 'msg_auction_restrictH1Bidder';
									SET @V_IsValidBid = 0;
									SET @V_RejectBidStatus  = 18;
								END

								IF(@V_IsValidBid = 1) /* IF bid is valid then check for multiple condition.  */
								BEGIN
									SET @V_OutputStr = 'msg_aucbid_success';
									SET @V_IsValidBid = 1;
									SET @V_RejectBidStatus  = 0;
									IF(@V_IncDecInMultiple = 1 AND (@V_CheckBidPrice - @V_H1L1Amount)%ABS(@V_IncDecAmount) ! =  0) /* multiple is yes AND  Bid is not valid in multiple of criteria?  */
									BEGIN
										SET @V_OutputStr = 'msg_aucbid_incrementmultiple';
										SET @V_IsValidBid = 0;
										SET @V_RejectBidStatus  = 3;
									END
								END
							END
							ELSE IF (@V_EvenTypeId = 2) /* Reverse Auction  */
							BEGIN
								/* Start - For Change Request #25679 - Increment/Decrement In times logic */
								IF(@V_IncDecType = 1 AND @V_IsBidPriceIncDecInTimesReq = 1 AND (@V_CheckBidPrice < (@V_H1L1Amount - (ABS(@V_IncDecAmount) * @V_BidPriceIncDecInTimes))))
								BEGIN
									SET @V_OutputStr = 'msg_aucbid_incdecintimes';
									SET @V_IsValidBid = 0;
									SET @V_RejectBidStatus  = 17;
								END /* End - For Change Request #25679 - Increment/Decrement In times logic */
								Else IF (((@V_StartPrice !=  0 AND @V_AuctionType = 2) OR (@V_AuctionType = 1)) AND @V_CheckBidPrice > @V_StartPrice) /* First bid condition = Auction Start Price and bid received greather than start price?  */
								BEGIN
									SET @V_OutputStr = 'msg_aucbid_decrementstartpricecriteria';
									SET @V_IsValidBid = 0;
									SET @V_RejectBidStatus  = 9;
								END
								ELSE IF (@V_IsFirstBid = 0 AND @V_FirstBidCond = 1 AND @V_CheckBidPrice < =  @V_H1L1Amount) /* First bid condition = Auction Start Price?  */
								BEGIN
									SET @V_IsValidBid = 1;
								END
								ELSE IF (@V_CheckBidPrice < =  (@V_H1L1Amount - ABS(@V_IncDecAmount)) AND @V_IsValidBid != 0)
								BEGIN									
									SET @V_IsValidBid = 1;
									IF(@V_BidDate <= @V_StartDate AND @V_BidderId != @V_SessionUserId)
									BEGIN
										SELECT @V_Count = COUNT(1) FROM appauctionbid.tbl_AuctionBidRank
										WHERE auctionId = @V_AuctionId AND tableId = @V_TableId AND rowId = @V_RowId AND bidderId = @V_BidderId AND isActive = 1;
										IF(@V_Count != 0)
										BEGIN
											SET @V_IsValidBid = 0;
										END
									END
								END
								ELSE /*IF(@V_AuctionResult <> 3)*/
								BEGIN
									SET @V_OutputStr = 'msg_aucbid_decrementcriteria';
									SET @V_IsValidBid = 0;
									SET @V_RejectBidStatus  = 2;
								END
								
								/* Start - For CR:23766 To have check on minimum quantity required */
								 IF (@V_AuctionResult = 2 AND @V_isMinQtyChkReq = 1)
								 BEGIN
									IF (@V_QtyBidPrice < @V_minQtyBid)
									BEGIN
										SET @V_OutputStr = 'msg_auc_bid_minqtycheck';
										SET @V_IsValidBid = 0;
										SET @V_RejectBidStatus  = 15;
									END
									ELSE IF (@V_QtyBidPrice > @V_maxQtyBid)
									BEGIN
										SET @V_OutputStr = 'msg_auc_bid_maxqtycheck';
										SET @V_IsValidBid = 0;
										SET @V_RejectBidStatus  = 16;
									END
								 END
								 /* End - For CR:23766 To have check on minimum quantity required */
								/* Add here emd validation check */
								
								IF (@V_IsValidBid = 1 AND ((@V_EvenTypeId = 1 AND @V_IsEmdReq = 1 AND (@V_IsBidCapacityReq = 1 OR @V_IsBidCapacityReq = 2)) OR (@V_EvenTypeId = 2 AND @V_IsEmdReq = 1 AND @V_IsBidCapacityReq = 2 AND @V_IssecurityFees = 2 AND @V_AuctionResult = 2 AND @V_emdPaymentMode = 1))) /* If emd is enabled and Bid Capacity required as Yes or EMD wise and Auction is Forward or Reverse*/
								BEGIN
									IF(@V_IsBidCapacityReq = 1)
									BEGIN
										IF ((@V_CheckBidPrice / @V_BiddingCapacity) > @V_EmdBalance)
										BEGIN
											/* Reject bid because of EMD balance is sufficient with respect to bid */
											SET @V_OutputStr = 'msg_aucbid_emdinsufficient';
											SET @V_IsValidBid = 0;
											SET @V_RejectBidStatus  = 21;
										END
									END
									/* Reject bid because of EMD balance is sufficient with respect to bid FOR V_IsBidCapacityReq == 2*/
									ELSE IF(@V_IsBidCapacityReq = 2) 
									BEGIN
										IF(@V_EmdBalance < @V_ConfigEmdAmt)
										BEGIN
											SET @V_OutputStr = 'msg_auc_payemd';
											SET @V_IsValidBid = 0;
											SET @V_RejectBidStatus  = 21;
										END
									END
								END

								/* Indrajit Maheshwari - EMD Rollout - Insufficient EMD Balance Check */
								IF(@V_IsValidBid = 1 AND @V_IsEmdReq = 2 AND @V_isRestrictBidderOnl1h1EMD = 1 AND @V_AuctionResult = 2 AND @V_emdPaymentMode = 1)
								BEGIN
									IF(@V_EmdBalance < @V_ConfigEmdAmt AND @V_IsPvsH1Bidder != 1)
										BEGIN
											SET @V_OutputStr = 'msg_aucbid_insufficient_emd_balance';
											SET @V_IsValidBid = 0;
											SET @V_RejectBidStatus  = 28;
									END
								END
								/* EMD Rollout - Insufficient EMD Balance Check END*/
								
								/* If Auctrion is Item wise, Standard, Open, IsRestrictH1Bidder is Yes and Bidder is Rank 1 in 1 Item*/
								IF(@V_AuctionResult = 2 AND @V_AuctionStatus = 1 AND @V_IsRestrictH1Bidder =1 AND @V_AuctionMode = 2 AND @V_ItemH1Count = 1) 
								BEGIN
									SET @V_OutputStr = 'msg_auction_restrictH1Bidder';
									SET @V_IsValidBid = 0;
									SET @V_RejectBidStatus  = 18;
								END

								IF(@V_IsValidBid = 1) /* IF bid is valid then check for multiple condition.  */
								BEGIN
									SET @V_OutputStr = 'msg_aucbid_success';
									SET @V_IsValidBid = 1;
									SET @V_RejectBidStatus  = 0;
									IF(@V_IncDecInMultiple = 1 AND (@V_H1L1Amount - @V_CheckBidPrice)%ABS(@V_IncDecAmount) ! =  0) /* multiple criteria is Yes And Bid is not valid in multiple of criteria?  */
									BEGIN
										SET @V_OutputStr = 'msg_aucbid_decrementmultiple';
										SET @V_IsValidBid = 0;
										SET @V_RejectBidStatus  = 3;
									END
								END
							END

							/* Restrict Bid by Captcha - Start - Indrajit Maheshwari */
							SELECT @V_ClientID = DEPT.clientId FROM appauction.tbl_Auction AC inner join appclient.tbl_Department DEPT ON AC.deptId=DEPT.deptId WHERE AC.auctionId = @V_AuctionId
							SELECT @V_IsRestrictByCaptcha=isRestrictByCaptcha, @V_CaptchaRestrictEnableDurationInSec=captchaRestrictEnableDurationInSec FROM appclient.tbl_Client WHERE clientId = @V_ClientID;
							IF(@V_IsRestrictByCaptcha = 1)
							BEGIN
								SELECT @V_IsFirstBid = COUNT(1) FROM appauctionbid.tbl_AuctionBidRank
									WHERE auctionId = @V_AuctionId AND tableId = @V_TableId AND bidderId = @V_BidderId AND isActive = 1;

								IF(@V_IsFirstBid != 0 AND @V_BidType = 0 AND @v_isAutoBidAllowed = 0)
								BEGIN 
									SELECT TOP 1 @V_LastBidTimestamp = submittedOn FROM appauctionbid.tbl_AuctionBidRank
										WHERE auctionId = @V_AuctionId AND tableId = @V_TableId AND bidderId = @V_BidderId AND isActive = 1 ORDER BY submittedOn DESC; --AND rowId = @V_RowId
									

									IF(@V_IsValidCaptcha = 0 AND DATEDIFF(ss, @V_LastBidTimestamp, GETUTCDATE()) <= @V_CaptchaRestrictEnableDurationInSec)
									BEGIN
										SET @V_OutputStr = 'msg_aucbid_restrictbycaptcha';
										SET @V_IsValidBid = 0;
										SET @V_RejectBidStatus  = 99; /* 99 - Captcha Restrict Status - New - Indrajit Maheshwari*/
									END
								END
							END
							/* Restrict Bid by Captcha - End - Indrajit Maheshwari */

						END /* END OF (Start price is not 0 or is not first bid)  */
					END /*IF(@V_IsValidBid ! =  0)  */
				END	/* END OF IF(@V_RejectBidStatus = 0)  */
			 END /* End of Else for IF(@V_BidType = 1) ==> Auto Bid */
			
		END /* END OF IF (@V_BidPrice ! =  0.0)  */
		ELSE
		BEGIN
			SET @V_OutputStr = 'msg_aucbid_zerobid';
			SET @V_IsValidBid = 0;
			SET @V_RejectBidStatus  = 4;
		END	
			
		IF(@V_RejectBidStatus NOT IN (5,6,7,8,11,14,99))
		BEGIN		
			/* take existing rank of bidder for extension check condition */
			IF(@V_RankForExt > 0)
			BEGIN
				SELECT @V_OldRank  = bidderRank FROM appauctionbid.tbl_AuctionBidRank WHERE bidderId = @V_BidderId 
				AND rowId = @V_RowId AND tableId = @V_TableId AND auctionId = @V_AuctionId AND isActive = 1;
			END	
			
			IF(@V_BiddingType = 2 AND @V_BidderCurrencyId != @V_BaseCurrencyId)/* FOR ICB - If Bidder's currency is not same as auction base currency?*/
			BEGIN
				SELECT @V_CheckBidPrice = appauction.F_CurrencyConvertion(@V_BidPrice, @V_BidderCurrencyId, @V_BaseCurrencyId, @V_BaseCurrencyId, @V_AuctionId, @V_DecimalValueUpto, @V_EvenTypeId, 0);
			END
						
			IF (@V_BidType = 0) /* IF bid type  = mannual */
			BEGIN			
				/***Step:6 Make entry into bid tables ***/
				/* Making entry for tbl_AucitonBid  */
				INSERT INTO appauctionbid.tbl_AuctionBid
				(auctionId,bidderId,userDetailId,tableId,rowId,bidPrice,bidderBidPrice,ipAddress,isAutoBid,submittedBy,submittedOn,isApproved,rejectStatus)
				VALUES
				(@V_AuctionId,@V_BidderId,@V_UserDetailId,@V_TableId,@V_RowId,@V_CheckBidPrice,@V_BidPrice,@V_IPAddress,@V_BidType,@V_SessionUserId,@V_BidDate,@V_IsValidBid,@V_RejectBidStatus);
				SELECT  @V_BidId = SCOPE_IDENTITY();
				
				
				/* Making entry for tbl_AuctionBidRank If Bidder bid is valid  */
				IF(@V_IsValidBid = 1 AND @V_BidId ! =  0) /* IF Valid bid then make enty into AuctionBidRank  */
				BEGIN
					
					/* Make entry in rank table if auction is both and @V_IsDisplayL1ItemWiseAndGTWise */

					SELECT @V_Count = COUNT(1) FROM appauctionbid.tbl_AuctionBidRank  
					WHERE auctionId = @V_AuctionId AND tableId = @V_TableId AND rowId = @V_RowId AND bidderId = @V_BidderId;


					IF(@V_Count = 0) 
					BEGIN

						IF(@V_AuctionResult = 3)
						BEGIN
							SET @V_DelimIndex = CHARINDEX(',', @V_TempCheckBidAmtStr, 0);
							WHILE (@V_TempCheckBidAmtStr !=  '')
							BEGIN
								IF(@V_DelimIndex !=  0)
								BEGIN
									SET @V_TempBidPrice = CONVERT(DECIMAL(20,5),SUBSTRING(@V_TempCheckBidAmtStr, 0, @V_DelimIndex));
									SET @V_TempCheckBidAmtStr = SUBSTRING(@V_TempCheckBidAmtStr, @V_DelimIndex + 1, LEN(@V_TempCheckBidAmtStr) - @V_DelimIndex);
								END
								ELSE
								BEGIN
									SET @V_TempBidPrice = CONVERT(DECIMAL(20,5),@V_TempCheckBidAmtStr);
									SET @V_TempCheckBidAmtStr  = '';
								END
								IF(@V_BiddingType = 2 AND @V_BidderCurrencyId != @V_BaseCurrencyId)/* FOR ICB - If Bidder's currency is not same as auction base currency?*/
								BEGIN
									SELECT @V_TempBidPrice = appauction.F_CurrencyConvertion(@V_TempBidPrice, @V_BidderCurrencyId, @V_BaseCurrencyId, @V_BaseCurrencyId, @V_AuctionId, @V_DecimalValueUpto, @V_EvenTypeId, 0);
								END
								INSERT INTO appauctionbid.tbl_AuctionBidRank 
								(auctionId,bidderId,tableId,rowId,bidId,bidPrice,bidQty,qtyAllocated,bidderRank,submittedOn,submittedBy,isActive) values
								(@V_AuctionId,@V_BidderId,@V_TableId,@V_TempRowId,@V_BidId,@V_TempBidPrice,0,0,1,@V_BidDate,@V_SessionUserId,1);

								SET @V_TempRowId = @V_TempRowId + 1;
								SET @V_DelimIndex = CHARINDEX(',', @V_TempCheckBidAmtStr, 0);
							END
						END
						
						INSERT INTO appauctionbid.tbl_AuctionBidRank
						(auctionId,bidderId,tableId,rowId,bidId,bidPrice,bidderRank,submittedOn,submittedBy,isActive)
						VALUES
						(@V_AuctionId,@V_BidderId,@V_TableId,@V_RowId,@V_BidId,@V_CheckBidPrice,1,@V_BidDate,@V_SessionUserId,1);
					END
					ELSE
					BEGIN
						IF(@V_AuctionResult = 3)
						BEGIN
							SET @V_DelimIndex = CHARINDEX(',', @V_CheckBidAmtStr, 0);
							WHILE (@V_TempCheckBidAmtStr !=  '')
							BEGIN
								IF(@V_DelimIndex !=  0)
								BEGIN
									SET @V_TempBidPrice = CONVERT(DECIMAL(20,5),SUBSTRING(@V_TempCheckBidAmtStr, 0, @V_DelimIndex));
									SET @V_TempCheckBidAmtStr = SUBSTRING(@V_TempCheckBidAmtStr, @V_DelimIndex + 1, LEN(@V_TempCheckBidAmtStr) - @V_DelimIndex);
								END
								ELSE
								BEGIN
									SET @V_TempBidPrice = CONVERT(DECIMAL(20,5),@V_TempCheckBidAmtStr);
									SET @V_TempCheckBidAmtStr  = '';
								END

								IF(@V_BiddingType = 2 AND @V_BidderCurrencyId != @V_BaseCurrencyId)/* FOR ICB - If Bidder's currency is not same as auction base currency?*/
								BEGIN
									SELECT @V_TempBidPrice = appauction.F_CurrencyConvertion(@V_TempBidPrice, @V_BidderCurrencyId, @V_BaseCurrencyId, @V_BaseCurrencyId, @V_AuctionId, @V_DecimalValueUpto, @V_EvenTypeId, 0);
								END
								
								UPDATE appauctionbid.tbl_AuctionBidRank
								SET bidPrice = @V_TempBidPrice,bidId = @V_BidId,submittedBy = @V_SessionUserId,isActive = 1,submittedOn = @V_BidDate
								WHERE auctionId = @V_AuctionId AND tableId = @V_TableId AND rowId = @V_TempRowId AND bidderId = @V_BidderId;

								SET @V_TempRowId = @V_TempRowId + 1;
								SET @V_DelimIndex = CHARINDEX(',', @V_TempCheckBidAmtStr, 0);
							END
						END
						UPDATE appauctionbid.tbl_AuctionBidRank
						SET bidPrice = @V_CheckBidPrice,bidId = @V_BidId,submittedBy = @V_SessionUserId,isActive = 1,submittedOn = @V_BidDate
						WHERE auctionId = @V_AuctionId AND tableId = @V_TableId AND rowId = @V_RowId AND bidderId = @V_BidderId;
					END
					
					SELECT @V_Count = COUNT(1) FROM appauctionbid.tbl_AuctionBidRank  
					WHERE auctionId = @V_AuctionId AND tableId = @V_TableId AND rowId = @V_RowId AND isActive = 1;
					
					/* Update rank of bidder's  */
					IF(@V_EvenTypeId = 1) /* Forward auction  */
					BEGIN
					
						SET @V_QueryStr = 'UPDATE T
										SET bidderRank = S.bidderRank
										FROM appauctionbid.tbl_AuctionBidRank T 
										INNER JOIN 
										(
											SELECT TOP '+CONVERT(VARCHAR,@V_Count)+' bidRankId,bidderId,bidPrice,ROW_NUMBER() OVER (ORDER BY bidPrice DESC,submittedOn ASC,bidId ASC) As bidderRank FROM appauctionbid.tbl_AuctionBidRank
											WHERE auctionId = @V_AuctionId AND tableId = @V_TableId AND rowId = @V_RowId AND isActive = 1
											ORDER BY bidPrice DESC,submittedOn ASC,bidId ASC
										) S
										on T.bidRankId = S.bidRankId';
					END
					ELSE /* Reverse Auction  */
					BEGIN
						SET @V_QueryStr = 'UPDATE T
										SET bidderRank = S.bidderRank
										FROM appauctionbid.tbl_AuctionBidRank T 
										INNER JOIN 
										(
											SELECT TOP '+CONVERT(VARCHAR,@V_Count)+' bidRankId,bidderId,bidPrice,ROW_NUMBER() OVER (ORDER BY bidPrice ASC,submittedOn ASC,bidId ASC) As bidderRank FROM appauctionbid.tbl_AuctionBidRank
											WHERE auctionId = @V_AuctionId AND tableId = @V_TableId AND rowId = @V_RowId AND isActive = 1
											ORDER BY bidPrice ASC,submittedOn ASC,bidId ASC
										) S
										on T.bidRankId = S.bidRankId';
					END
					print @V_QueryStr
					EXECUTE SP_EXECUTESQL @V_QueryStr,
					N'@V_Count int,@V_AuctionId int,@V_TableId int,@V_RowId int',
					@V_Count = @V_Count,@V_AuctionId = @V_AuctionId	,@V_TableId = @V_TableId,@V_RowId = @V_RowId	;


					IF(@V_AuctionResult = 3)
					BEGIN
						SET @V_DelimIndex = CHARINDEX(',', @V_CheckBidAmtStr, 0);
						SET @V_TempCheckBidAmtStr = @V_CheckBidAmtStr;
						SET @V_TempRowId = 1;

						WHILE (@V_TempCheckBidAmtStr !=  '')
						BEGIN
							IF(@V_DelimIndex !=  0)
							BEGIN
								SET @V_TempBidPrice = CONVERT(DECIMAL(20,5),SUBSTRING(@V_TempCheckBidAmtStr, 0, @V_DelimIndex));
								SET @V_TempCheckBidAmtStr = SUBSTRING(@V_TempCheckBidAmtStr, @V_DelimIndex + 1, LEN(@V_TempCheckBidAmtStr) - @V_DelimIndex);
							END
							ELSE
							BEGIN
								SET @V_TempBidPrice = CONVERT(DECIMAL(20,5),@V_TempCheckBidAmtStr);
								SET @V_TempCheckBidAmtStr  = '';
							END
							EXECUTE SP_EXECUTESQL @V_QueryStr,
							N'@V_Count int,@V_AuctionId int,@V_TableId int,@V_RowId int',
							@V_Count = @V_Count,@V_AuctionId = @V_AuctionId	,@V_TableId = @V_TableId,@V_RowId = @V_TempRowId	;

							SET @V_TempRowId = @V_TempRowId + 1;
							SET @V_DelimIndex = CHARINDEX(',', @V_TempCheckBidAmtStr, 0);
						END
					END

					
					/* Update EMD Balance of all bidders If emd is enabled and Bidding capacity required as yes and if auction is Forward - Start */
					IF(@V_EvenTypeId = 1 AND @V_IsEmdReq = 1 AND @V_IsBidCapacityReq = 1)
					BEGIN
					
						/* If standard and forward auction with emd as yes then EMD insert emd utilise entry*/
						SET @V_EmdBalance = @V_EmdBalance - (@V_CheckBidPrice / @V_BiddingCapacity)
						INSERT INTO appauction.tbl_EmdUtilise (emdId,emdAmtUsed,emdBalance,createdOn,createdBy,updatedOn,updatedBy,status)
							VALUES(@V_EmdBalId,(@V_CheckBidPrice / @V_BiddingCapacity),@V_EmdBalance, @V_BidDate, @V_SessionUserId, @V_BidDate, @V_SessionUserId,1)

						/* Get All H1 Bidder Ids */
						DECLARE CurEmdCalc CURSOR LOCAL FAST_FORWARD FOR
						SELECT DISTINCT(bidderId) FROM appauctionbid.tbl_AuctionBidRank WHERE auctionId = @V_AuctionId AND isActive = 1
						
						OPEN CurEmdCalc;
						FETCH NEXT FROM CurEmdCalc INTO @V_bidderIdEmd
						WHILE (@@Fetch_status = 0)							
						BEGIN
							/* Get Toal H1 Bid Price of H1 Bidder */
							SELECT @V_totalH1Bid = ISNULL(SUM(bidPrice),0) FROM appauctionbid.tbl_AuctionBidRank 
							WHERE auctionId = @V_AuctionId AND bidderId = @V_bidderIdEmd AND isActive = 1  AND bidderRank = 1
							
							/* Update Emd balance of H1 Bidder - Start */
							IF (@V_totalH1Bid != 0)
							BEGIN
								SET @V_EmdUsed = ROUND(@V_totalH1Bid / @V_BiddingCapacity, @V_DecimalValueUpto) 
								
								UPDATE appauction.tbl_EmdBalance SET emdBalance = (emdAmount - @V_EmdUsed), updatedOn = @V_BidDate, updatedBy = @V_SessionUserId 
									WHERE auctionId = @V_AuctionId AND bidderId = @V_bidderIdEmd
								
							END
							ELSE
							BEGIN
								UPDATE appauction.tbl_EmdBalance SET emdBalance = emdAmount, updatedOn = @V_BidDate, updatedBy = @V_SessionUserId  
									WHERE auctionId = @V_AuctionId AND bidderId = @V_bidderIdEmd								
							END
							/* Update Emd balance of H1 Bidder - End */
							
						FETCH NEXT FROM CurEmdCalc INTO @V_bidderIdEmd
						END
						CLOSE CurEmdCalc; 
						DEALLOCATE CurEmdCalc;
					END						
					ELSE IF((@V_EvenTypeId = 1 AND @V_IsEmdReq = 1 AND @V_IsBidCapacityReq = 2) OR (@V_EvenTypeId = 2 AND @V_IsEmdReq = 1 AND @V_IsBidCapacityReq = 2 AND @V_IssecurityFees = 2 AND @V_AuctionResult = 2 AND @V_emdPaymentMode = 1))
					BEGIN
						SELECT @V_H2bidder = count(*) FROM appauctionbid.tbl_AuctionBidRank WHERE auctionId = @V_AuctionId AND isActive = 1 AND bidderRank = 2  AND rowId = @V_RowId
						
						/* If standard and forward auction with emd as yes then EMD insert emd utilise entry*/
						SET @V_EmdBalance = (@V_EmdBalance - @V_ConfigEmdAmt)
						
							INSERT INTO appauction.tbl_EmdUtilise (emdId,emdAmtUsed,emdBalance,createdOn,createdBy,updatedOn,updatedBy,status)
							VALUES(@V_EmdBalId,(@V_ConfigEmdAmt),@V_EmdBalance, @V_BidDate, @V_SessionUserId, @V_BidDate, @V_SessionUserId,1)
						
						/* Get All H1 Bidder Ids */
						DECLARE CurEmdCalc CURSOR LOCAL FAST_FORWARD FOR
						SELECT DISTINCT(bidderId) FROM appauctionbid.tbl_AuctionBidRank WHERE auctionId = @V_AuctionId AND isActive = 1 AND rowId = @V_RowId
						
						OPEN CurEmdCalc;
						FETCH NEXT FROM CurEmdCalc INTO @V_bidderIdEmd
						WHILE (@@Fetch_status = 0)							
						BEGIN
							/*Get EMD Balance , Amount and Rank of all bidder in cursor*/
							SELECT  @V_EmdBalance = emdBalance, @V_EmdAmount = emdAmount FROM appauction.tbl_EmdBalance WHERE auctionId = @V_AuctionId AND bidderId = @V_bidderIdEmd
							SELECT @V_IsRankExits = count(*) FROM appauctionbid.tbl_AuctionBidRank WHERE auctionId = @V_AuctionId AND isActive = 1  AND bidderRank > 2  AND rowId = @V_RowId AND bidderId = @V_bidderIdEmd
							
							/* Get Toal H1 Bid Price count of H1 Bidder */
							SELECT @V_totalH1Bid = count(*) FROM appauctionbid.tbl_AuctionBidRank 
							WHERE auctionId = @V_AuctionId AND bidderId = @V_bidderIdEmd AND isActive = 1  AND bidderRank = 1 AND rowId = @V_RowId
							
							/* Update Emd balance of H1 Bidder - Start */
							IF (@V_totalH1Bid != 0)
							BEGIN
								/* Allow bidder to bid if its 1st time or not H1 of particular Item */
								IF(@V_IsPvsH1Bidder != 1 AND @V_bidderIdEmd = @V_BidderId)
								BEGIN
										UPDATE appauction.tbl_EmdBalance SET emdBalance = (emdBalance - @V_ConfigEmdAmt), updatedOn = @V_BidDate, updatedBy = @V_SessionUserId 
										WHERE auctionId = @V_AuctionId AND bidderId = @V_bidderIdEmd
								END
								ELSE
								BEGIN
									BREAK
								END
							END
							/* Give return amount of H2 Bidder */
							ELSE IF(@V_bidderIdEmd != @V_BidderId AND @V_H2bidder = 1 AND @V_IsPvsH1Bidder = 0 AND ((@V_EmdBalance + @V_ConfigEmdAmt) <= @V_EmdAmount) AND @V_IsRankExits <> 1)
							BEGIN
								UPDATE appauction.tbl_EmdBalance SET emdBalance = (emdBalance + @V_ConfigEmdAmt), updatedOn = @V_BidDate, updatedBy = @V_SessionUserId 
								WHERE auctionId = @V_AuctionId AND bidderId = @V_bidderIdEmd	
							END
							/* Update Emd balance of H1 Bidder - End */
							
						FETCH NEXT FROM CurEmdCalc INTO @V_bidderIdEmd
						END
						CLOSE CurEmdCalc; 
						DEALLOCATE CurEmdCalc;
					END
					print @v_isAutoBidAllowed;
					/* Update EMD Balance of all bidders If emd is enabled and if auction is standard - End */
					IF(@v_isAutoBidAllowed = 1) /* Auto bid allowed */
					BEGIN
						EXEC appauctionbid.p_AutoBid @v_AuctionId,@V_TableId,@V_RowId,@V_IPAddress,@V_SessionUserId,@V_UserDetailId,@V_BidDate,@V_CalledFrom,@V_IsValidAutoBid;
					END

					/* Indrajit Maheshwari - EMD Rollout - Update EMD Balance of all bidders If Auction & EMD is Itemwise and Restrict Bidder On L1/H1 EMD is Yes and EMD Payment Mode is Online(PG) */
					IF(@V_IsValidBid = 1 AND @V_IsEmdReq = 2 AND @V_isRestrictBidderOnl1h1EMD = 1 AND @V_AuctionResult = 2 AND @V_emdPaymentMode = 1)
					BEGIN
						SELECT @V_H2bidder = count(*) FROM appauctionbid.tbl_AuctionBidRank WHERE auctionId = @V_AuctionId AND isActive = 1 AND bidderRank = 2  AND rowId = @V_RowId
	
						SET @V_EmdBalance = (@V_EmdBalance - @V_ConfigEmdAmt)
	
						/* Get All H1 Bidder Ids */
						DECLARE CurEmdCalc CURSOR LOCAL FAST_FORWARD FOR
						SELECT DISTINCT(bidderId) FROM appauctionbid.tbl_AuctionBidRank WHERE auctionId = @V_AuctionId AND isActive = 1 AND rowId = @V_RowId
	
						OPEN CurEmdCalc;
						FETCH NEXT FROM CurEmdCalc INTO @V_bidderIdEmd
						WHILE (@@Fetch_status = 0)							
						BEGIN
							/*Get EMD Balance , Amount and Rank of all bidder in cursor*/
							SELECT @V_EmdBalance = emdBalance, @V_EmdAmount = emdAmount FROM appauction.tbl_EmdBalance WHERE auctionId = @V_AuctionId AND bidderId = @V_bidderIdEmd
							SELECT @V_IsRankExits = count(*) FROM appauctionbid.tbl_AuctionBidRank WHERE auctionId = @V_AuctionId AND isActive = 1  AND bidderRank > 2  AND rowId = @V_RowId AND bidderId = @V_bidderIdEmd
		
							/* Get Total H1 Bid Price count of H1 Bidder */
							SELECT @V_totalH1Bid = count(*) FROM appauctionbid.tbl_AuctionBidRank WHERE auctionId = @V_AuctionId AND bidderId = @V_bidderIdEmd AND isActive = 1  AND bidderRank = 1 AND rowId = @V_RowId
		
							/* Update Emd balance of H1 Bidder - Start */
							IF (@V_totalH1Bid != 0)
							BEGIN
								/* Allow bidder to bid if its 1st time or not H1 of particular Item */
								IF(@V_IsPvsH1Bidder != 1 AND @V_bidderIdEmd = @V_BidderId)
								BEGIN
										INSERT INTO appauction.tbl_EmdUtilise (emdId,emdAmtUsed,emdBalance,createdOn,createdBy,updatedOn,updatedBy,status)
										VALUES(@V_EmdBalId,@V_ConfigEmdAmt,@V_EmdBalance, @V_BidDate, @V_SessionUserId, @V_BidDate, @V_SessionUserId,1)

										UPDATE appauction.tbl_EmdBalance SET emdBalance = (emdBalance - @V_ConfigEmdAmt), updatedOn = @V_BidDate, updatedBy = @V_SessionUserId 
										WHERE auctionId = @V_AuctionId AND bidderId = @V_bidderIdEmd
								END
								ELSE
								BEGIN
									BREAK
								END
							END
							/* Update Emd balance of H1 Bidder - End */
		
							/* Give return amount of H2 Bidder */
							/* EMDBalanceBidderID != BidBidderId and H2Bidder exists and BidBidder is not previous H1 Bidder AND (total emd balance+item emd <= bidder total emd amount and rank > 2 not exists) */
							ELSE IF(@V_bidderIdEmd != @V_BidderId AND @V_H2bidder = 1 AND @V_IsPvsH1Bidder = 0 AND ((@V_EmdBalance + @V_ConfigEmdAmt) <= @V_EmdAmount) AND @V_IsRankExits <> 1)
							BEGIN
								INSERT INTO appauction.tbl_EmdUtilise (emdId,emdAmtUsed,emdBalance,createdOn,createdBy,updatedOn,updatedBy,status)
								VALUES(@V_EmdBalId,@V_ConfigEmdAmt,@V_EmdBalance, @V_BidDate, @V_SessionUserId, @V_BidDate, @V_SessionUserId,1)
							
								UPDATE appauction.tbl_EmdBalance SET emdBalance = (emdBalance + @V_ConfigEmdAmt), updatedOn = @V_BidDate, updatedBy = @V_SessionUserId 
								WHERE auctionId = @V_AuctionId AND bidderId = @V_bidderIdEmd	
							END
		
						FETCH NEXT FROM CurEmdCalc INTO @V_bidderIdEmd
						END
						CLOSE CurEmdCalc; 
						DEALLOCATE CurEmdCalc;
					END
					/* EMD Rollout END*/

				END	
				
				/* Making entry for tbl_AuctionBidDetail  */
				EXECUTE SP_EXECUTESQL @V_BidDetailStr, 	N'@V_BidId int', @V_BidId = @V_BidId;
				
			
			END /* IF (@V_BidType = 0) -- IF bid type  = mannual */
			

			/* IF IsValidBid and AUTO BID OUTBIDDED - Indrajit Maheshwari - START */
			SELECT @V_ClientID = DEPT.clientId FROM appauction.tbl_Auction AC inner join appclient.tbl_Department DEPT ON AC.deptId=DEPT.deptId WHERE AC.auctionId = @V_AuctionId
			BEGIN TRY
				IF(@V_IsValidBid = 1)
				BEGIN
					/* Get All H2 Bidder Ids with Auto Bid */
					DECLARE CurAutoCalc CURSOR LOCAL FAST_FORWARD FOR
					SELECT DISTINCT ABR.bidderId, AB.initialPrice, AB.bestPrice, AB.updatedOn FROM appauctionbid.tbl_AuctionBidRank ABR 
						INNER JOIN appauctionbid.tbl_AutoBid AB
						ON ABR.auctionId=AB.auctionId AND ABR.bidderId=AB.bidderId AND ABR.rowId=AB.rowId
						WHERE ABR.auctionId=@V_AuctionId AND ABR.isActive = 1 AND bidderRank = 2  AND ABR.rowId = @V_RowId

					OPEN CurAutoCalc;
					FETCH NEXT FROM CurAutoCalc INTO @V_BidderIdAutoBid, @V_LowerLimit, @V_UpperLimit, @V_AutoBidConfigOn
					WHILE (@@Fetch_status = 0)							
					BEGIN
					
						SELECT @V_EmailId = loginid	FROM appuser.tbl_UserLogin where userId = @V_BidderIdAutoBid

						SELECT @V_Subject = DMC.subject, @V_MsgBoxSubject = DMC.messageSubject, @V_Body = REPLACE(CONVERT(NVARCHAR(MAX), DMC.mailContent),'$emailId',@V_EmailId) 
							+ CONVERT(NVARCHAR(MAX), DMC.signature), @V_MsgBoxBody = REPLACE(CONVERT(NVARCHAR(MAX), DMC.messageContent),'$emailId',@V_EmailId) 
							+ CONVERT(NVARCHAR(MAX), DMC.signature), @V_FromAddress = DMC.fromId, @V_LinkId = DMC.linkId   
							FROM appmaster.tbl_DynMailContent DMC WHERE DMC.mailTemplateId = 434;
					
						SET @V_TableContents = '<br>
							<table border="1" cellpadding="1" cellspacing="1">
							<tr>
								<th>Sr. No.</th>
								<th>Auction ID</th>
								<th>';
									IF (@V_AuctionResult = 2)
									BEGIN
										SET @V_TableContents = @V_TableContents + 'Item Name';
									END
									ELSE
									BEGIN
										SET @V_TableContents = @V_TableContents + 'Auction Brief';
									END
					
									SET @V_TableContents = @V_TableContents + '
								</th>
								<th>Configured Auto Bid Amount</th>
								<th>Date and Time of Configured Auto Bid</th>
							</tr>';
					

						IF (@V_AuctionResult = 2)
						BEGIN
							SELECT @V_ItemName = tblauctioncell.cellValue  
								FROM appauction.tbl_AuctionTable tblauctiontable 
								INNER JOIN appauction.tbl_AuctionColumn tblauctioncolumn ON tblauctiontable.tableId=tblauctioncolumn.tableId and tblauctioncolumn.columnTypeId = 1 
								INNER JOIN appauction.tbl_AuctionCell tblauctioncell ON tblauctioncell.columnId = tblauctioncolumn.columnId
								WHERE tblauctiontable.auctionId = @V_AuctionId AND tblauctioncell.rowId = @V_RowId AND tblauctiontable.cstatus = 1 
								ORDER BY tblauctioncell.rowId 

						END
						ELSE
						BEGIN
							SET @V_ItemName = @V_ItemName + @V_AuctionBrief;
						END

						IF (@V_EvenTypeId = 1)
						BEGIN
							SET @V_TableLimits = @V_TableLimits + @V_UpperLimit;
						END
						ELSE
						BEGIN
							SET @V_TableLimits = @V_TableLimits + @V_LowerLimit;
						END


						SET @V_TableContents = @V_TableContents + '
							<tr>
								<td>1</td>
								<td>' + CAST(@V_AuctionId as NVARCHAR(10)) + '</td>
								<td>' + CAST(@V_ItemName as NVARCHAR(MAX)) + '</td>
								<td>' + CAST(CAST(@V_TableLimits AS numeric(20, 2)) as NVARCHAR(50)) + '</td>
								<td>' + CAST(appmaster.F_ToDateTime(@V_AutoBidConfigOn, '+05:30', '103') as NVARCHAR(50)) + '</td>
							</tr>';

						SET @V_TableContents = @V_TableContents + '</table>';
						    		    
						SET @V_Body = REPLACE(@V_Body, '$TableDetails',@V_TableContents);
						SET @V_Subject = REPLACE(@V_Subject, '$EventID',@V_AuctionId);

						/* Send mail to bidder to notify about his outbid for Auto Bid configured	 */
						EXEC msdb.dbo.sp_send_dbmail @profile_name = 'eProcSMTP', @recipients = @V_EmailId,
						@blind_copy_recipients='',
						@subject = @V_Subject, @body = @V_Body, @body_format = 'HTML', @from_address = @V_FromAddress;
						/* Send messageBox mail to bidder - START */
						SELECT @V_IsMsgBoxMailRequired = byMessageBox from appclient.tbl_ClientAlertConf where linkId = @V_LinkId and clientId = @V_ClientID;
						IF (@V_IsMsgBoxMailRequired = 1)
						BEGIN
							SET @V_MsgBoxBody = REPLACE(@V_MsgBoxBody, '$TableDetails',@V_TableContents);
							SET @V_MsgBoxSubject = REPLACE(@V_MsgBoxSubject, '$EventID',@V_AuctionId);
							INSERT INTO [appuser].[tbl_Message] ([clientId],[subject],[messageText],[messageFrom],[userTypeId],[parentMessageId],[emailId],[isDocUpload],[actionType],[createdOn],[createdBy],[cstatus])
							VALUES(@V_ClientID, @V_MsgBoxSubject, @V_Body, 0, 2, 0, @V_FromAddress, 0, 1, GETUTCDATE(), @V_BidderIdAutoBid, 1)
							INSERT INTO [appuser].[tbl_MessageDetail] ([messageTo],[messageId],[emailId],[folderId],[sendType],[isMessageRead],[updatedOn],[updatedBy])
							VALUES(@V_BidderIdAutoBid, SCOPE_IDENTITY(), @V_EmailId, 1, 1, 0, GETUTCDATE(), @V_BidderIdAutoBid)
						END
						/* Send messageBox mail to bidder - END */
			
						/* Insert data into tbl_AlertHistory */
						INSERT INTO appreport.tbl_AlertHistory(alertType, alertId, linkId, alertFrom, alertTo, subject, sentDate, clientId) 
						VALUES (1, NEWID(), @V_LinkId, @V_FromAddress, @V_EmailId, @V_Subject, GETUTCDATE(), @V_ClientID);


						FETCH NEXT FROM CurAutoCalc INTO @V_BidderIdAutoBid, @V_LowerLimit, @V_UpperLimit, @V_AutoBidConfigOn
					END
					CLOSE CurAutoCalc; 
					DEALLOCATE CurAutoCalc;
				END
				END TRY
				BEGIN CATCH
					BEGIN
						INSERT INTO appreport.tbl_ExceptionLog
						(errorMessage,linkId,fileName,className,method,lineNumber,createdOn,createdBy)
						SELECT ISNULL(ERROR_MESSAGE(),'-'),0,'p_BidSubmission','-','AutoBid Outbided Notification',ERROR_LINE(),GETUTCDATE(),0;
					END
				END CATCH
			/* IF IsValidBid and AUTO BID OUTBIDDED - Indrajit Maheshwari - END */


			/***Step:7 In case of Valid bid then Check any Extension Need to be triggered ***/
			IF(@V_IsValidBid = 1 AND @V_RankForExt > 0)
			BEGIN
				SELECT @V_Count  = bidderRank FROM appauctionbid.tbl_AuctionBidRank WHERE bidderId = @V_BidderId 
				AND rowId = @V_RowId AND tableId = @V_TableId AND auctionId = @V_AuctionId AND isActive = 1;
			END
			
			SELECT @V_EndDateVirtual = endDateVirtual,@V_CurrentExt = currentExt,@V_RankForExt = rankForExt,@V_ExtendBy = extendBy,@V_ExtendWhen = extendWhen,@V_TotalExt = totalExt 
			FROM appauction.tbl_Auction	WHERE auctionId = @V_AuctionId;
			
			IF(@V_IsItemWiseTime = 1) /* If Item wise time required = Yes */
			BEGIN
				SELECT 	@V_EndDateVirtual = endDateVirtual,@V_IsAutoExt = isAutoExt,
				@V_ExtMode = extMode,@V_ExtendBy = extendBy,@V_ExtendWhen = extendWhen,@V_TotalExt = totalExt,@V_CurrentExt = currentExt
				FROM appauction.tbl_AuctionCriteria WHERE auctionId = @V_AuctionId AND rowId = @V_RowId;
			END

			IF(@V_IsValidBid = 1 AND @V_IsAutoExt = 1 AND (@V_RankForExt = 0 OR (@V_Count <=  @V_RankForExt AND @V_OldRank != @V_Count)) AND (@V_BidDate BETWEEN  DATEADD(MI,-(@V_ExtendWhen),@V_EndDateVirtual) AND @V_EndDateVirtual))
			BEGIN
					IF(@V_ExtMode = 1 AND @V_CurrentExt > =  @V_TotalExt) /* IF limited extension and all extensions are finished?  */
					BEGIN
						PRINT('All extensions are finished');
					END
					ELSE 
					BEGIN
						IF (@V_ExtMode = 1) /* Fixed Extension  */
						BEGIN
							Set @V_EndDateVirtual = DATEADD(MI,@V_ExtendBy,@V_EndDateVirtual) ;
						END
						ELSE IF (@V_ExtMode = 2) /* Unlimited Extension  */
						BEGIN
							Set @V_EndDateVirtual = DATEADD(MI,@V_ExtendBy,@V_BidDate) ;	
						END
						
						IF(  @V_AuctionResult = 2 AND @V_IsItemWiseTime = 1 AND @v_configureTimeForItem != 2) /* If Item wise time required = Yes and Not Serial Auction  */
						BEGIN

							SELECT @V_ExtMode = extMode,@V_TotalExt = totalExt,@V_CurrentExt = currentExt
							FROM appauction.tbl_AuctionCriteria WHERE auctionId = @V_AuctionId AND rowId = @V_RowId;
							
							UPDATE appauction.tbl_AuctionCriteria
							SET endDateVirtual = @V_EndDateVirtual,
							currentExt = CASE WHEN (@V_ExtMode = 1 AND currentExt < totalExt AND @V_CurrentExt = currentExt) then (@V_CurrentExt + 1)
							ELSE  CASE WHEN (@V_ExtMode = 2) then @V_CurrentExt + 1 END END  
							WHERE auctionId = @V_AuctionId AND rowId = @V_RowId;							
							
							SELECT @V_EndDateVirtual = MAX(endDateVirtual) FROM appauction.tbl_AuctionCriteria
							WHERE auctionId = @V_AuctionId
							GROUP BY auctionId;
							
							UPDATE appauction.tbl_Auction 
							SET endDateVirtual = @V_EndDateVirtual
							WHERE auctionId = @V_AuctionId;
						END
						ELSE IF(  @V_AuctionResult = 2 AND @V_IsItemWiseTime = 1  AND @v_configureTimeForItem = 2) /* If Item wise time required = Yes and Serial Auction  */
						BEGIN
							
							SELECT @V_ExtMode = extMode,@V_TotalExt = totalExt,@V_CurrentExt = currentExt
							FROM appauction.tbl_AuctionCriteria WHERE auctionId = @V_AuctionId AND rowId = @V_RowId;

							SELECT @V_SerialEndDateVirtual = endDateVirtual FROM appauction.tbl_Auction
							WHERE auctionId = @V_AuctionId							
							
							IF (@V_ExtMode = 1) /* Fixed Extension  */
							BEGIN
								Set @V_SerialEndDateVirtual = DATEADD(MI,@V_ExtendBy,@V_SerialEndDateVirtual) ;
							END
							ELSE IF (@V_ExtMode = 2) /* Unlimited Extension  */
							BEGIN
								Set @V_SerialEndDateVirtual = DATEADD(MI,@V_ExtendBy,@V_BidDate) ;	
							END
							
							UPDATE appauction.tbl_AuctionCriteria
							SET endDateVirtual = @V_EndDateVirtual,
							currentExt = CASE WHEN (@V_ExtMode = 1 AND currentExt < totalExt AND @V_CurrentExt = currentExt) then (@V_CurrentExt + 1)
							ELSE  CASE WHEN (@V_ExtMode = 2) then @V_CurrentExt + 1 END END 
							WHERE auctionId = @V_AuctionId AND rowId = @V_RowId;							
							
							UPDATE appauction.tbl_Auction 
							SET endDateVirtual = @V_SerialEndDateVirtual
							WHERE auctionId = @V_AuctionId;
						END
						ELSE
						BEGIN
							
							SELECT @V_ExtMode = extMode,@V_CurrentExt = currentExt,@V_TotalExt = totalExt 
							FROM appauction.tbl_Auction	WHERE auctionId = @V_AuctionId;
							
							UPDATE appauction.tbl_AuctionCriteria
							SET endDateVirtual = @V_EndDateVirtual,
							currentExt = CASE WHEN (@V_ExtMode = 1 AND currentExt < totalExt AND @V_CurrentExt = currentExt) then (@V_CurrentExt + 1)
							ELSE  CASE WHEN (@V_ExtMode = 2) then @V_CurrentExt + 1 END END 
							WHERE auctionId = @V_AuctionId;						
							
							UPDATE appauction.tbl_Auction
							SET endDateVirtual = @V_EndDateVirtual,
							currentExt = CASE WHEN (@V_ExtMode = 1 AND currentExt < totalExt AND @V_CurrentExt = currentExt) then (@V_CurrentExt + 1)
							ELSE  CASE WHEN (@V_ExtMode = 2) then @V_CurrentExt + 1 END END 
							WHERE auctionId = @V_AuctionId ;
						END
					END
			END	/* end of IF(@V_IsValidBid = 1 AND @V_IsAutoExt = 1 AND (@V_RankForExt = 0 OR (@V_Count <=  @V_RankForExt AND @V_OldRank != @V_Count)) AND (@V_BidDate BETWEEN  DATEADD(MI,-(@V_ExtendWhen),@V_EndDateVirtual) AND @V_EndDateVirtual)) */
			
		END	 /* END OF IF(@V_RejectBidStatus NOT IN (5,6,7,8))  */
		SELECT @V_bidSubmittedDate = @V_BidDate;
		--SELECT @V_OutputStr returnStr;
		COMMIT TRANSACTION;
		
    END TRY
    BEGIN CATCH
		BEGIN
			SET @V_OutputStr = 'msg_bidsubmission_exception';
			ROLLBACK TRANSACTION;
			INSERT INTO appreport.tbl_ExceptionLog
			(errorMessage,linkId,fileName,className,method,lineNumber,createdOn,createdBy,clientId)
            SELECT ISNULL(ERROR_MESSAGE(),'-'),0,'p_BidSubmission','-','-',ERROR_LINE(),GETUTCDATE(),@V_SessionUserId,1;

		END
    END CATCH
END

------------------------------------------------------------------------------------------------------------------------------------




