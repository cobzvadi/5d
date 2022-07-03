//@version=5
//@author=LucF

indicator("Realtime 5D Profile [LucF]", "VPAA", true, max_labels_count = 500, precision = 8)

// "Realtime 5D Profile [LucF]
//  v6, 2022.01.02 13:09 — LucF

// This indicator displays a realtime profile using five dimensions: volume, price, time, activity and age.

// The code was written using the Pine Style Guide recommendations: https://www.tradingview.com/pine-script-docs/en/v5/writing/Style_guide.html

// What this script does is log the time, price, volume and polarity of each chart update in `rt*_` arrays, which have one element per rt update.
// Current data for the profile is held in `squash*_ arrays, which contain two (+/-) subtotals per levels for timestamps, volumes and ticks, from which displayed profile level values will then be calculated.
// As updates come in, if the profile was not rescaled because price breached the profile's hi/lo, 
// updates are simply merged into current `squash*_` arrays because there is no need to re-squash all logged rt updates for the profile into `squash*_` arrays.
// If the profile is rescaled, then we must re-squash to account for the new distribution of updates in the profile's new levels.
// Once `squash*_` arrays are updated, we can process their values to redraw the required profile(s) on each update.



// ———————————————————— Constants {

// Base colors.
var color BLUE         = #013BCAff
var color BLUE_LITE    = #013BCA20
var color GOLD         = #CCCC00ff
var color GRAY         = #808080ff
var color GRAY_LITE    = #80808020
var color GREEN        = #008000ff
var color LIME         = #00FF00ff
var color PINK         = #FF0080ff
var color RED          = #FF0000ff
var color YELLOW       = #FFFF00ff

// ————— Constants used in inputs.
// Options.
var string PROF1 = "    🠇 Volume 🠅"
var string PROF2 = "  🠇 Ticks 🠅"
var string PROF3 = " Ticks | Volume"
var string PROF4 = "Volume | Ticks"
var string PROF5 = "Volume |"
var string PROF6 = " Ticks |"
var string PROF7 = "        | Volume"
var string PROF8 = "        | Ticks"

var string RSET1 = "Timeframe"
var string RSET2 = "Supertrend"
var string RSET3 = "LazyBear's Weis Wave Trend Detection"

var string PFMT1 = "Total"
var string PFMT2 = "Delta"

var string CAND0 = "None"
var string CAND1 = "Polarity of update"
var string CAND2 = "Polarity of close vs open"
var string CAND3 = "Trend"

// Default colors.
var color  UP_COLOR = LIME
var color  DN_COLOR = PINK
var color  UP_COLOR_TOTAL = GREEN
var color  DN_COLOR_TOTAL = RED

// Tooltips.
var string TT_CONFIG    = "DOUBLE-SIDED PROFILES\nThe first two configurations are called 'double-sided' profiles. With them, the settings in this section control both sides.\n\n" +
                          "SINGLE-SIDED PROFILES\nThe other configurations use 'single-sided' profiles. With those, the formatting choices in this section control the LEFT side of the profile, and the settings in the following section control its RIGHT side."
var string TT_DURATION  = "Determines when a new profile is created. Use this when you don't want a new profile on each bar. You can change the profile on a higher timeframe or on trend changes. The 'Timeframe' setting has no effect when using trend-detection logic.\n\n" +
                          "See the 'Trend Detection and Display' section further down for settings related to trend detection."
var string TT_LEVELS    = "'Levels' (1 to 100) determines the maximum quantity of levels in the profile. If the profile's height is small enough for level increments to be smaller than minticks, then a smaller quantity of levels are used until the profile's height grows.\n\n" +
                          "The offset allows shifting the profile left/right by a −/+ number of bars."
var string TT_CALC      = "This only applies when you are using single-sided profiles.\With 'Total', level values are the total volume/ticks at that level for the bar.\n" +
                          "With 'Delta', level values represent the net values for the level (up values - down values)."
var string TT_SIZE      = "'Max line length' (0 to 100) controls the length of lines. Use zero if you only need values.\n'Profile size' controls the overall size of the profile."

// ————— Constants used as arguments in `profileBuild*()` and `profileDraw*()` calls.
var bool   RIGHT_PROFILE = true
var bool   LEFT_PROFILE  = false
var bool   VOLUME = true
var bool   TICKS  = false
// }



// ———————————————————— Inputs {

string  profileConfigInput          = input.string(PROF1,       "Configuration",                inline = "00", options = [PROF1, PROF2, PROF3, PROF4, PROF5, PROF6, PROF7, PROF8], tooltip = TT_CONFIG)
string  profileResetOnTfInput       = input.string(RSET1,       "Resets on",                    inline = "01", options = [RSET1, RSET2, RSET3])
string  profileResetTfInput         = input.timeframe("",       "Timeframe",                    inline = "01", tooltip = TT_DURATION)
int     profileLevelsMaxQtyInput    = input.int(20,             "Levels",                       inline = "02", minval = 1, maxval = 100)
int     profileOffsetInput          = input.int(0,              " ↔ Offset (in bars)",          inline = "02", tooltip = TT_LEVELS)
int     profileVolumePrecisionInput = input.int(2,              "Precision of volume values",   inline = "02a", minval = 0)

bool    profileLeftCalcDeltaInput   = input.string(PFMT2,       "Calculation",                  inline = "03", options = [PFMT1, PFMT2], tooltip = TT_CALC) == PFMT2
color   profileLeftBullColorInput   = input(UP_COLOR,           "Level Colors 🠅",              inline = "04")
color   profileLeftBearColorInput   = input(DN_COLOR,           "🠇",                           inline = "04")
color   profileLeftBullTColorInput  = input(UP_COLOR_TOTAL,     "  Total Colors 🠅",            inline = "04")
color   profileLeftBearTColorInput  = input(DN_COLOR_TOTAL,     "🠇",                           inline = "04")
int     profileLeftMaxWidthInput    = input.int(15,             "Maximum line length",          inline = "05", minval = 0, maxval = 100)
string  profileLeftSizeInput        = input.string(size.small,  "Profile size",                 inline = "05", options = [size.tiny, size.small, size.normal, size.large, size.huge], tooltip = TT_SIZE)
bool    profileLeftShowValuesInput  = input.bool(true,          "Level values  ",               inline = "06")
bool    profileLeftShowTotalsInput  = input.bool(true,          "Total bar values using size",  inline = "06")
string  profileLeftTotalsSizeInput  = input.string(size.normal, "",                             inline = "06", options = [size.tiny, size.small, size.normal, size.large, size.huge])

var GRP1 = "🠆 Right format (single-sided profiles only)"
bool    profileRightCalcDeltaInput  = input.string(PFMT2,       "Calculation",                  inline = "11", group = GRP1, options = [PFMT1, PFMT2], tooltip = TT_CALC) == PFMT2
color   profileRightBullColorInput  = input(UP_COLOR,           "Level Colors 🠅",              inline = "12", group = GRP1)
color   profileRightBearColorInput  = input(DN_COLOR,           "🠇",                           inline = "12", group = GRP1)
color   profileRightBullTColorInput = input(UP_COLOR_TOTAL,     "  Total Colors 🠅",            inline = "12", group = GRP1)
color   profileRightBearTColorInput = input(DN_COLOR_TOTAL,     "🠇",                           inline = "12", group = GRP1)
int     profileRightMaxWidthInput   = input.int(15,             "Maximum line length",          inline = "13", group = GRP1, minval = 0, maxval = 100)
string  profileRightSizeInput       = input.string(size.small,  "Profile size",                 inline = "13", group = GRP1, options = [size.tiny, size.small, size.normal, size.large, size.huge], tooltip = TT_SIZE)
bool    profileRightShowValuesInput = input.bool(true,          "Show values  ",                inline = "14", group = GRP1)
bool    profileRightShowTotalsInput = input.bool(true,          "Show totals using size",       inline = "14", group = GRP1)
string  profileRightTotalsSizeInput = input.string(size.normal, "",                             inline = "14", group = GRP1, options = [size.tiny, size.small, size.normal, size.large, size.huge])

var GRP2 = "Candles"
string  candleBorderModeInput       = input.string(CAND0,       "Color borders on",             inline = "21", group = GRP2, options = [CAND0, CAND1, CAND2, CAND3])
color   candleBorderBullColorInput  = input(GRAY,               " 🠅",                          inline = "21", group = GRP2)
color   candleBorderBearColorInput  = input(BLUE,               "🠇",                           inline = "21", group = GRP2)
string  candleBodyModeInput         = input.string(CAND0,       "Color bodies on ",             inline = "22", group = GRP2, options = [CAND0, CAND1, CAND2, CAND3])
color   candleBodyBullTColorInput   = input(GRAY_LITE,          " 🠅",                          inline = "22", group = GRP2)
color   candleBodyBearTColorInput   = input(BLUE_LITE,          "🠇",                           inline = "22", group = GRP2)
bool    candleColoringRtInput       = input.bool(true,          "On RT bars only",              inline = "23", group = GRP2)

var GRP3 = "Trend Detection and Display"
int     trendSTLengthInput          = input.int(20,             "Supertrend length",            inline = "31", group = GRP3, minval = 1)
float   trendSTFactorInput          = input.float(4.0,          " Factor",                      inline = "31", group = GRP3, minval = 0.0)
int     trendWWSmoothingLengthInput = input.int(2,              "LB Weis Wave smoothing",       inline = "32", group = GRP3, minval = 1)
bool    trendMarkerShowChangesInput = input.bool(true,          "Show trend changes",           inline = "33", group = GRP3)
bool    trendMarkerChangesRtInput   = input.bool(true,          "On RT bars only",              inline = "33", group = GRP3)
color   trendMarkerBullColorInput   = input(UP_COLOR,           " 🠅",                          inline = "33", group = GRP3)
color   trendMarkerBearColorInput   = input(DN_COLOR,           "🠇",                           inline = "33", group = GRP3)
bool    trendMarkerShowStatesInput  = input.bool(true,          "Show trend state    ",         inline = "34", group = GRP3)
bool    trendMarkerStatesRtInput    = input.bool(true,          "On RT bars only",              inline = "34", group = GRP3)
// }



// ———————————————————— Global data {

// Convert profile configuration and formats to bool states so tests are faster when building profile than if we compared strings.
var bool profileDOUBLE_VOLUME = profileConfigInput == PROF1
var bool profileDOUBLE_TICKS  = profileConfigInput == PROF2
var bool profileLEFT_VOLUME   = profileConfigInput == PROF4 or profileConfigInput == PROF5
var bool profileLEFT_TICKS    = profileConfigInput == PROF3 or profileConfigInput == PROF6
var bool profileRIGHT_VOLUME  = profileConfigInput == PROF3 or profileConfigInput == PROF7
var bool profileRIGHT_TICKS   = profileConfigInput == PROF4 or profileConfigInput == PROF8

var bool profileResetOnTF = profileResetOnTfInput == RSET1
var bool profileResetOnST = profileResetOnTfInput == RSET2
var bool profileResetOnWW = profileResetOnTfInput == RSET3

// Arrays containing one element per realtime chart update for the current profile, to be squashed into profile levels when the profile is rescaled. 
// Used by `squashRtUpdates()` and `getRtUpdate()`.
varip int[]   rtTimes_   = array.new_int()
varip float[] rtPrices_  = array.new_float()
varip float[] rtVolumes_ = array.new_float()
varip bool[]  rtUps_     = array.new_bool()

// Arrays of values for levels, obtained from squashing rt values. 
// Reset on each recalc of profile levels by `profileRecalcLevels()`. Updated by `squashRtUpdates()`.
// Their IDs are passed as arguments to profile building and drawing functions.
varip float[] squashedVolumesUp_ = array.new_float()
varip float[] squashedVolumesDn_ = array.new_float()
varip int[]   squashedTicksUp_   = array.new_int()
varip int[]   squashedTicksDn_   = array.new_int()
// Holds the average time of rt updates in squashed arrays. Used to determine the age of a level.
varip float[] squashedTimesUp_   = array.new_float()
varip float[] squashedTimesDn_   = array.new_float()

// Arrays holding strings used for levels. Used by functions `profileInitLevelStrings()` and `profileGetLevelStr()`.
var string[] levelStringHeight1_ = array.new_string()
var string[] levelStringHeight2_ = array.new_string()
var string[] levelStringHeight3_ = array.new_string()
var string[] levelStringHeight4_ = array.new_string()

// Array of prices corresponding to each profile level. It rescales dynamically when a new hi or lo is found for the profile's price range, 
// which may include changing its size if the qty of levels changes in the profile. Function `profileRecalcLevels()` handles the maintenance of this array.
varip float[] profilePrices_ = array.new_float()
// Bar index where current profile is drawn.
varip int profileBarIndex_ = na
// Hi/lo points of current profile.
varip float profileHi_ = low
varip float profileLo_ = high
// When `true`, a new profile must be created.
varip bool beginNewProfile_ = true
// When `true`, the profile rescales and levels are recaclculated.
varip bool rescaleProfile_ = true
// Qty of rt updates logged for the current profile.
varip int qtyOfRtUpdates_ = 0
// Qty of rt updates logged for the current profile.
varip float qtyOfRtVolumes_ = 0.0
// Qty of levels in the current profile.
varip int qtyOfLevels_ = array.size(profilePrices_)
// Decimal precision used to display volume values.
var string volumePrecision_ = "#" + (profileVolumePrecisionInput == 0 ? "" : ".") + array.join(array.new_string(profileVolumePrecisionInput, "#"))
// `true` when tick is up.
var bool[] tickUp = array.new_bool(1, false)
// Maximum user width of profiles.
var int profileMaxWidth_ = math.max(profileLeftMaxWidthInput, profileRightMaxWidthInput)
// }



// ———————————————————— Functions {

printMessage(series string txt, series color bgColor = color.yellow) => 
    //@function         Displays a `_txt` string with a `_bgcolor` in bottom right of the indicator's visual space.
    //@param txt        String to display.
    //@param bgColor    Background color used for the message.
    //@returns          Nothing.
    
    var table t = table.new(position.bottom_right, 1, 1)
    if barstate.islast
        table.cell(t, 0, 0, txt, text_color = color.white, bgcolor = bgColor)


tfInMinutes(simple string tf = "") => 
    //@function     Converts the `tf` timeframe to a float minutes value.
    //@param tf     The timeframe string in `timeframe.period` format to be converted to float minutes. If no argument is supplied, the chart's timeframe is used.
    //@returns      A float value in minutes, e.g., 1440.0 for "D", or 0.5 for 30sec timeframes.

    float tfUnitsInSeconds = 
      switch
        timeframe.isseconds => 1. / 60
        timeframe.isminutes => 1.
        timeframe.isdaily   => 60. * 24
        timeframe.isweekly  => 60. * 24 * 7
        timeframe.ismonthly => 60. * 24 * 30.4375
    float tfInSeconds = timeframe.multiplier * tfUnitsInSeconds
    float result = tf == "" ? tfInSeconds : request.security(syminfo.tickerid, tf, tfInSeconds)


nonRepaintingSecurity(simple string sym, simple string tf, src) =>
    //@function     Returns a non-repainting value with `request.security()`. Cannot be used with functions returning tuples.
    //@param        Parameters are the same as `request.security()`: symbol, timeframe and source series.
    //@returns      A non-repainting value that behaves the same way on historical and realtime bars.
    request.security(sym, tf, src[barstate.isrealtime ? 1 : 0])[barstate.isrealtime ? 0 : 1]


wwLBTrend(series float src, simple int length) =>
    //Thanks to LazyBear. Original code: https://www.tradingview.com/script/HFGx4ote-Indicator-Weis-Wave-Volume-LazyBear/
    //@function     Detects price reversals using a smoothing confirmation.
    //@param src    Source used for price.
    //@param length Smoothing length.
    //@returns       +1/-1 for an up/dn trend.
    var int latestTrend = 0
    var int prevTrend = 0
    int priceChange = int(math.sign(ta.change(src)))
    latestTrend := priceChange and ta.change(priceChange) ? priceChange : latestTrend
    bool wwConfirmation = ta.rising(src, length) or ta.falling(src, length)
    prevTrend := (latestTrend != prevTrend and wwConfirmation) ? latestTrend : prevTrend


profileInitLevelStrings(simple int maxWidth) =>
    //@function         Builds the full set of strings containing all the possible quantities of the repeated character producing each of 4 possible line heights. These strings are what the labels will be printing for each profile level.
    //@param maxWidth   The maximum quantity of characters for which to build lines. This has an impact on the range of line lengths and the width of the profile.
    //@returns          Nothing.
    //Changes:          levelStringHeight*_
    
    // Characters used to build lines of 4 different heights.
    var string CHAR_1 = "▂" //U+9602
    var string CHAR_2 = "▃" //U+9603
    var string CHAR_3 = "▅" //U+9605
    var string CHAR_4 = "▆" //U+9606 

    for qtyOfCharsInLine = 0 to maxWidth
        string lineOfChars1 = array.join(array.new_string(qtyOfCharsInLine, CHAR_1))
        string lineOfChars2 = array.join(array.new_string(qtyOfCharsInLine, CHAR_2))
        string lineOfChars3 = array.join(array.new_string(qtyOfCharsInLine, CHAR_3))
        string lineOfChars4 = array.join(array.new_string(qtyOfCharsInLine, CHAR_4))
        array.push(levelStringHeight1_, lineOfChars1)
        array.push(levelStringHeight2_, lineOfChars2)
        array.push(levelStringHeight3_, lineOfChars3)
        array.push(levelStringHeight4_, lineOfChars4)


profileGetLevelStr(series int width, series int height = 1) =>
    //@param width  Line's width in 0 to `profileMaxWidth_` range.
    //@param height Line's height in 1 to 4 range.
    //@returns  The level string containing `width` characters of `height` height.
    
    string result = 
      switch height
        1 => array.get(levelStringHeight1_, width)
        2 => array.get(levelStringHeight2_, width)
        3 => array.get(levelStringHeight3_, width)
        4 => array.get(levelStringHeight4_, width)
        => ""


profileRecalcLevels(series float hi, series float lo, simple int userQtyOfLevels) =>
    //@function              Recalculate quantity of levels and increments of profile levels, adjust user-defined qty if necessary.
    //@param hi              The high price of the profile.
    //@param lo              The low price of the profile.
    //@param userQtyOfLevels The user-selected qty of profile levels. This will be the actual qty of levels unless the profile's vertical range is too narrow.
    //@returns               The new quantity of levels in the profile.
    //Changes:               profilePrices_, squashed*_
    //Dependencies:          *_, *Input
    
    // ————— Determine qty of levels.
    // Profiles always have at least one level, even when their height is zero.
    float profileHeight     = math.abs(hi - lo)
    int   tickLevels        = int(profileHeight / syminfo.mintick)
    // If the height of the profile does not allow for as many levels as user wants, calculate the max number of levels that `syminfo.mintick` will allow.
    int   levels            = math.max(1, profileHeight / (userQtyOfLevels) < syminfo.mintick ? tickLevels : userQtyOfLevels)
    float profileIncrements = profileHeight / levels

    // ————— Rebuild the array containing the price level corresponding to each profile level and squash arrays.
    array.clear(profilePrices_)
    array.clear(squashedVolumesUp_)
    array.clear(squashedVolumesDn_)
    array.clear(squashedTicksUp_)
    array.clear(squashedTicksDn_)
    array.clear(squashedTimesUp_)
    array.clear(squashedTimesDn_)
    // When there is only one level, use the middle of the range.
    if levels == 1
        array.push(profilePrices_, math.avg(hi, lo))
        array.push(squashedVolumesUp_, 0.0)
        array.push(squashedVolumesDn_, 0.0)
        array.push(squashedTicksUp_,   0)
        array.push(squashedTicksDn_,   0)
        array.push(squashedTimesUp_,   0.0)
        array.push(squashedTimesDn_,   0.0)
    else
        for levelNo = 0 to levels - 1
            array.push(profilePrices_, lo + ((0.5 + levelNo) * profileIncrements))
            array.push(squashedVolumesUp_, 0.0)
            array.push(squashedVolumesDn_, 0.0)
            array.push(squashedTicksUp_,   0)
            array.push(squashedTicksDn_,   0)
            array.push(squashedTimesUp_,   0.0)
            array.push(squashedTimesDn_,   0.0)

    // Return new qty of levels.
    levels


profileRemovePrevious(series int barIndex) =>
    //@function         Deletes last drawn profile.
    //@param barIndex   Bar index where we want to delete the previously drawn profile.
    //@returns          Nothing.

    while array.size(label.all) > 0
        label lbl = array.pop(label.all)
        int labelX = label.get_x(lbl)
        if labelX == barIndex
            label.delete(lbl)
        else
            break


sortTwoArrays(keysArray, valuesArray, order = order.ascending) =>
    //Thx to @Duyck for the original source of this function: https://www.tradingview.com/script/Mzc4dmq7-ma-sorter-sort-by-array-example-JD/
    //@function          Sorts array `arrayToSort` while keeping array `arrayKeys` in synch with the sorted order.
    //@param arrayKeys   Array of keys allowing us to identify sorted results.
    //@param arrayToSort Array of values to sort. All values must be unique.
    //@returns           IDs of the two sorted arrays.
    
    qtyOfValues       = array.size(keysArray)
    arrayKeysSorted   = array.copy(keysArray)
    valuesArraySorted = array.copy(valuesArray)
    if qtyOfValues == array.size(valuesArray)
        array.sort(valuesArraySorted, order)
        for i = 0 to qtyOfValues - 1
            sortedValue   = array.get(valuesArraySorted, i)
            unsortedIndex = array.indexof(valuesArray, sortedValue)
            unsortedKey   = array.get(keysArray, unsortedIndex)
            array.set(arrayKeysSorted, i, unsortedKey)
    
    [arrayKeysSorted, valuesArraySorted]


profileBuildAgeOfLevels(series int[] leftHeights, series int[] rightHeights) =>
    //@function             Calculates the average age of each level and sets values in `heights` array.
    //@param leftHeights    Array of line 1 to 4 heights representing the age of each dn level, where 4 represents the most recent age.
    //@param rightHeights   Same for up levels.
    //@returns              Nothing.
    //Changes:              leftHeights, rightHeights
    //Dependencies:         *_, *Input

    // Keyed arrays holding the avg timestamp values for each up/dn level.
    float[] avgAgesDn  = array.new_float()
    float[] avgAgesUp  = array.new_float()
    int[]   levelNosDn = array.new_int()
    int[]   levelNosUp = array.new_int()
    for levelNo = 0 to qtyOfLevels_ - 1
        // Create our keys arrays.
        array.push(levelNosDn, levelNo)
        array.push(levelNosUp, levelNo)
        // Get avg of the total of all timestamps to determine the level's age.
        ticksInLevelDn = array.get(squashedTicksDn_, levelNo)
        ticksInLevelUp = array.get(squashedTicksUp_, levelNo)
        float avgDn = nz(array.get(squashedTimesDn_, levelNo) / ticksInLevelDn)
        float avgUp = nz(array.get(squashedTimesUp_, levelNo) / ticksInLevelUp)
        // To prepare values for sorting with `sortTwoArrays()`, which requires unique values, replace zero values with a unique small fraction that will keep it at the bottom of sorted values.
        array.push(avgAgesDn, avgDn == 0 ? 0.001 * levelNo : avgDn)
        array.push(avgAgesUp, avgUp == 0 ? 0.001 * levelNo : avgUp)
        
    // Sort ages in descending avg age order.
    [sortedLevelNosDn, sortedAgesDn] = sortTwoArrays(levelNosDn, avgAgesDn, order.ascending)
    [sortedLevelNosUp, sortedAgesUp] = sortTwoArrays(levelNosUp, avgAgesUp, order.ascending)
    
    // ————— Assign line height corresponding to the quarter of the age of each level.
    int qtyOfZeroLevelsDn = 0
    int qtyOfZeroLevelsUp = 0
    int heightDn = na
    int heightUp = na
    for decreasingAge = 0 to qtyOfLevels_ - 1
        // Only attribute 1 to 4 height to non-zero levels, so need to calculate the age of non-zero levels relative to other non-zero levels only.
        float avgAgeDn = array.get(sortedAgesDn, decreasingAge)
        float avgAgeUp = array.get(sortedAgesUp, decreasingAge)
        if avgAgeDn < 1
            qtyOfZeroLevelsDn += 1
            heightDn := 0
        else
            // Calculate 1 to 4 height (4 for most recent levels), but only relative to current qty of non-zero levels.
            heightDn := math.max(1, nz(math.ceil(4 * (decreasingAge - qtyOfZeroLevelsDn) / (qtyOfLevels_ - qtyOfZeroLevelsDn - 1)), 4))
        if avgAgeUp < 1
            qtyOfZeroLevelsUp += 1
            heightUp := 0
        else
            // Calculate 1 to 4 height (4 for most recent levels), but only relative to current qty of non-zero levels.
            heightUp := math.max(1, nz(math.ceil(4 * (decreasingAge - qtyOfZeroLevelsUp) / (qtyOfLevels_ - qtyOfZeroLevelsUp - 1)), 4))
        
        array.set(leftHeights,  array.get(sortedLevelNosDn, decreasingAge), heightDn)
        array.set(rightHeights, array.get(sortedLevelNosUp, decreasingAge), heightUp)

        
profileBuildOneSide(series float[] values, series int[] widths, series color[] colors, simple bool doRightSide, simple bool doVolume, simple bool doDelta) =>
    //@function             Squashes realtime updates info at one element per tick into level arrays of values and colors.
    //@param squashedUp     Array of squashed volumes or ticks up.
    //@param values         Array of left profile values to be displayed.
    //@param widths         Array of left profile line width for each level.
    //@param colors         Array of left profile line color for each level.
    //@param doRightSide    `true` when we are building the right profile.
    //@param doVolume       `true` when we treat volume, `false` when it's ticks.
    //@param doDelta        `true` when we are calculation deltas. When it's `false`, we calculate gross values.
    //@returns              Nothing.
    //Changes:              values, widths, colors
    //Dependencies:         *_, *Input

    var color profileBullColor = doRightSide ? profileRightBullColorInput : profileLeftBullColorInput
    var color profileBearColor = doRightSide ? profileRightBearColorInput : profileLeftBearColorInput
    var int   profileMaxWidth  = doRightSide ? profileRightMaxWidthInput  : profileLeftMaxWidthInput
    var int   sign = doDelta ? -1 : +1
    
    squashedUp = doVolume ? squashedVolumesUp_ : squashedTicksUp_
    squashedDn = doVolume ? squashedVolumesDn_ : squashedTicksDn_
    
    // Calculate values along user-selected format
    float[] absDeltas = array.new_float(qtyOfLevels_)
    for levelNo = 0 to qtyOfLevels_ - 1
        float valueUp = array.get(squashedUp, levelNo)
        float valueDn = array.get(squashedDn, levelNo)
        array.set(values, levelNo, valueUp + (sign * valueDn))
        array.set(absDeltas, levelNo, math.abs(valueUp - valueDn))

    // ————— Determine line width and color of each level, for one side of the profile.
    float maxValueGross   = doVolume ? qtyOfRtVolumes_ : qtyOfRtUpdates_
    float maxValueProfile = array.max(absDeltas)
    float maxValue = doDelta ? maxValueProfile : maxValueGross
    
    for levelNo = 0 to qtyOfLevels_ - 1
        float value = array.get(values, levelNo)
        // Line width: proportional to max value in the profile's levels.
        int lineWidth = math.min(profileMaxWidth_, nz(int(math.ceil(profileMaxWidth * math.abs(value) / maxValue)), profileMaxWidth))
        array.set(widths, levelNo, lineWidth)
        
        // Line color: set bull color if the up value (volume or ticks) is greater than the dn value.
        color lineColor = value > 0 ? profileBullColor : profileBearColor
        array.set(colors, levelNo, lineColor)


profileBuildTwoSides(series int[] leftWidths, series color[] leftColors, series int[] rightWidths, series color[] rightColors, simple bool doVolume) =>
    //@function             Squashes realtime updates info at one element per tick into level arrays of values and colors.
    //@param leftWidths     Array of left profile line width for each level.
    //@param leftColors     Array of left profile line color for each level.
    //@param rightWidths    Array of right profile line width for each level.
    //@param rightColors    Array of right profile line color for each level.
    //@param doVolume       `true` when we treat volume, `false` when it's ticks.
    //@returns              Nothing.
    //Changes:              leftWidths, leftColors, rightWidths, rightColors
    //Dependencies:         *_, *Input
    
    squashedUp = doVolume ? squashedVolumesUp_ : squashedTicksUp_
    squashedDn = doVolume ? squashedVolumesDn_ : squashedTicksDn_

    // Determine line width, height, and color of each level, for each side of the profile.
    float rangeOfValues = math.max(array.max(squashedDn), array.max(squashedUp))
    for levelNo = 0 to qtyOfLevels_ - 1
        float valueDn = array.get(squashedDn, levelNo)
        float valueUp = array.get(squashedUp, levelNo)
        // Line width: proportional to max value in the profile's levels.
        int lineWidthDn = math.min(profileMaxWidth_, nz(int(math.ceil(profileLeftMaxWidthInput * valueDn / rangeOfValues)), profileLeftMaxWidthInput))
        int lineWidthUp = math.min(profileMaxWidth_, nz(int(math.ceil(profileLeftMaxWidthInput * valueUp / rangeOfValues)), profileLeftMaxWidthInput))
        array.set(leftWidths,  levelNo, lineWidthDn)
        array.set(rightWidths, levelNo, lineWidthUp)
        
        // Line color: set bull color if the up value (volume or ticks) is greater than the dn value.
        color lineColorDn = valueDn > valueUp ? profileLeftBearColorInput : color.new(profileLeftBearColorInput, 50)
        color lineColorUp = valueUp > valueDn ? profileLeftBullColorInput : color.new(profileLeftBullColorInput, 50)
        array.set(leftColors,  levelNo, lineColorDn)
        array.set(rightColors, levelNo, lineColorUp)
    

profileDrawOneSide(float[] values, series int[] widths, series int[] leftHeights, series int[] rightHeights, series color[] colors, simple bool doRightSide, simple bool doVolume) =>
    //@function           Draws a complete profile on the either side of the bar.
    //@param values       Array of values for each level (line widths correspond to these values).
    //@param widths       Array of line widths for each level.
    //@param leftHeights  Array of left side line heights for each level.
    //@param rightHeights Array of right side line heights for each level.
    //@param colors       Array of colors for each level.
    //@param doRightSide  `true` when we are building the right profile.
    //@param doVolume     `true` when we treat volume, `false` when it's ticks.
    //@returns            Nothing.
    // Dependencies:      *_, *Input
    
    var color  profileBullColor = doRightSide ? profileRightBullColorInput  : profileLeftBullColorInput
    var color  profileBearColor = doRightSide ? profileRightBearColorInput  : profileLeftBearColorInput
    var string profileStyle     = doRightSide ? label.style_label_left      : label.style_label_right
    var string profileTextalign = doRightSide ? text.align_left             : text.align_right
    var string profileSize      = doRightSide ? profileRightSizeInput       : profileLeftSizeInput
    var bool   profileShowVals  = doRightSide ? profileRightShowValuesInput : profileLeftShowValuesInput
    var bool   profileShowTots  = doRightSide ? profileRightShowTotalsInput : profileLeftShowTotalsInput
    var string profileTotsSize  = doRightSide ? profileRightTotalsSizeInput : profileLeftTotalsSizeInput
    var string valueSpacing     = (doRightSide ? profileRightMaxWidthInput  : profileLeftMaxWidthInput) == 0 ? "" : " "
    var string formatString     = valueSpacing + (doVolume ? volumePrecision_ : "#") + valueSpacing

    squashedUp = doVolume ? squashedVolumesUp_ : squashedTicksUp_
    squashedDn = doVolume ? squashedVolumesDn_ : squashedTicksDn_

    // Draw one label for each of the profile's levels. 
    for levelNo = 0 to qtyOfLevels_ - 1
        float  labelY       = array.get(profilePrices_, levelNo)
        float  value        = array.get(values, levelNo)
        string valueString  = profileShowVals and value != 0 ? str.tostring(value, formatString) : ""
        int    leftH        = array.get(leftHeights, levelNo)
        int    rightH       = array.get(rightHeights, levelNo)
        int    lineHeight   = int(math.ceil(leftH < 1 ? rightH : rightH < 1 ? leftH : math.avg(leftH, rightH)))
        string labelTxt     = (doRightSide ? "" : valueString) + profileGetLevelStr(array.get(widths, levelNo), lineHeight) + (doRightSide ? valueString : "")
        color  labelColor   = array.get(colors, levelNo)
        string labelTooltip = str.tostring(array.get(values, levelNo))
        label.new(profileBarIndex_, labelY, labelTxt, xloc.bar_index, yloc.price, color(na), profileStyle, labelColor, profileSize, profileTextalign, labelTooltip)

    // ————— Put totals on top.
    if profileShowTots
        varip float lastTotal = na
        var    profileTotal = doVolume ? qtyOfRtVolumes_ : qtyOfRtUpdates_
        float  total        = array.sum(values)
        float  labelY       = profileHi_ + 2 * (profileHi_ - array.get(profilePrices_, qtyOfLevels_ - 1))
        string labelTxt     = str.tostring(total, formatString) + (total > lastTotal ? " 🠅" : total < lastTotal ? " 🠇" : "")
        color  labelColor   = total >= 0 ? profileBullColor : profileBearColor
        string labelTooltip = str.tostring(total) + " / " + str.tostring(profileTotal)
        label.new(profileBarIndex_, labelY, labelTxt, xloc.bar_index, yloc.price, color(na), profileStyle, labelColor, profileTotsSize, profileTextalign, labelTooltip)
        
        // Save last value to detect change in next update.
        lastTotal := total


profileDrawTwoSides(series int[] leftWidths, series int[] leftHeights, series color[] leftColors, series int[] rightWidths, series int[] rightHeights, series color[] rightColors, simple bool doVolume) =>
    //@function             Draws a complete profile on the either side of the bar.
    //@param leftWidths     Array of left profile line width for each level.
    //@param leftHeights    Array of left profile line height for each level.
    //@param leftColors     Array of left profile line color for each level.
    //@param rightWidths    Array of right profile line width for each level.
    //@param rightHeights   Array of right profile line height for each level.
    //@param rightColors    Array of right profile line color for each level.
    //@param doVolume       `true` when we treat volume, `false` when it's ticks.
    //@returns              Nothing.
    //Dependencies:         *_, *Input

    var string valueSpacing = profileLeftMaxWidthInput == 0 ? "" : " "
    var string formatString = valueSpacing + (doVolume ? volumePrecision_ : "#") + valueSpacing
 
    // var string formatString = doVolume ? volumePrecision_ : " # "
    squashedUp = doVolume ? squashedVolumesUp_ : squashedTicksUp_
    squashedDn = doVolume ? squashedVolumesDn_ : squashedTicksDn_

    // ————— Draw one label for each of the profile's levels. 
    for levelNo = 0 to qtyOfLevels_ - 1
        // Left side.
        float  labelY       = array.get(profilePrices_, levelNo)
        float  value        = array.get(squashedDn, levelNo)
        string valueString  = profileLeftShowValuesInput and value != 0 ? str.tostring(value, formatString) : ""
        string labelTxt     = valueString + profileGetLevelStr(array.get(leftWidths, levelNo), array.get(leftHeights, levelNo))
        color  labelColor   = array.get(leftColors, levelNo)
        string labelTooltip = str.tostring(value)
        label.new(profileBarIndex_, labelY, labelTxt, xloc.bar_index, yloc.price, color(na), label.style_label_right, labelColor, profileLeftSizeInput, text.align_right, labelTooltip)

        // Right side.
        value        := array.get(squashedUp, levelNo)
        valueString  := profileLeftShowValuesInput and value != 0 ? str.tostring(value, formatString) : ""
        labelTxt     := profileGetLevelStr(array.get(rightWidths, levelNo), array.get(rightHeights, levelNo)) + valueString
        labelColor   := array.get(rightColors, levelNo)
        labelTooltip := str.tostring(value)
        label.new(profileBarIndex_, labelY, labelTxt, xloc.bar_index, yloc.price, color(na), label.style_label_left, labelColor, profileLeftSizeInput, text.align_left, labelTooltip)

    if profileLeftShowTotalsInput
        varip float lastDn = na
        varip float lastUp = na
        // ————— Put totals on top.
        // Get total up/dn values.
        float totalDn = array.sum(squashedDn)
        float totalUp = array.sum(squashedUp)
        // Draw left total.
        float  labelY       = profileHi_ + 2 * (profileHi_ - array.get(profilePrices_, qtyOfLevels_ - 1))
        string labelTxt     = (totalDn > lastDn ? "🠅 " : totalDn < lastDn ? "🠇 " : "") + str.tostring(totalDn, formatString)
        color  labelColor   = totalDn > totalUp ? profileLeftBearTColorInput : color.new(profileLeftBearTColorInput, 40)
        label.new(profileBarIndex_, labelY, labelTxt, xloc.bar_index, yloc.price, color(na), label.style_label_right, labelColor, profileLeftTotalsSizeInput, text.align_right)
        // Draw right total.
        labelTxt     := str.tostring(totalUp, formatString) + (totalUp > lastUp ? " 🠅" : totalUp < lastUp ? " 🠇" : "")
        labelColor   := totalUp > totalDn ? profileLeftBullTColorInput : color.new(profileLeftBullTColorInput, 40)
        label.new(profileBarIndex_, labelY, labelTxt, xloc.bar_index, yloc.price, color(na), label.style_label_left, labelColor, profileLeftTotalsSizeInput, text.align_left)

        // Save last value to detect change in next update.
        lastDn := totalDn
        lastUp := totalUp


getRtUpdate() =>
    //@function Calculates the volume and polarity of one realtime update.
    //@returns  New volume for the update.
    //Changes:  rt*_
    
    // ————— 1. Get polarity of update. When price has not moved since the last update, it uses the previous update's polarity.
    varip bool updUp = false
    varip bool updDn = false
    varip float prevClose = open
    bool flat = close == prevClose
    updUp := flat ? updUp : close > prevClose
    updDn := flat ? updDn : close < prevClose
    prevClose := close

    // ————— 2. Get volume of update.
    varip float lastVolume  = volume
    if barstate.isnew
        // New realtime bar or first realtime update when script loads on the chart; reset data.
        lastVolume  := 0.
    float newVolume = nz(volume) - lastVolume
    lastVolume := nz(volume)

    // Update global arrays.
    array.push(rtTimes_,   timenow)
    array.push(rtPrices_,  close)
    array.push(rtVolumes_, newVolume)
    array.push(rtUps_,     updUp)
    
    // Update global array element for use by plotting section.
    array.set(tickUp, 0, updUp)
    
    // Return new volume for the update.
    newVolume


squashRtUpdates(series bool profileWasRescaled) =>
    //@function                 Squashes realtime updates into `squash*` arrays containing one value for each profile level.
    //@param profileWasRescaled `true` when profile was rescaled, which entails we must rebuild the squash arrays from scratch.
    //@returns                  Nothing.
    //Dependencies:             *_
    //Changes:                  qtyOfLevels_, squashed*_

    // ————— Squash rt updates into profile level values.
    // If profile wasn't rescaled, only squash last value by making the `while` loop iterate only once on most recent rt update.
    int rtUpdateNo    = profileWasRescaled ? 0 : qtyOfRtUpdates_ - 1
    
    // Discover in which profile level each rt update fits and update that level.
    int levelToUpdate = 0
    while rtUpdateNo < qtyOfRtUpdates_
        priceToSlot = array.get(rtPrices_, rtUpdateNo)
        if qtyOfLevels_ > 1
            // Loop through profile prices starting from the bottom, looking for slot where the price of the rt entry fits. Profile levels are matched when value is between half below or half above the level.
            for levelNo = 0 to qtyOfLevels_ - 1
                if levelNo < qtyOfLevels_ - 1
                    // We are not yet at last profile level; check for a match. Values not matched here belong to last level.
                    float levelToMatch = math.avg(array.get(profilePrices_, levelNo), array.get(profilePrices_, levelNo + 1))
                    if priceToSlot <= levelToMatch
                        levelToUpdate := levelNo
                        break
                    int(na)
                else
                    // Last level and no slot was found; value belongs to last level.
                    levelToUpdate := levelNo
        
        // —————————— We have found the profile level where this value goes; update squash arrays with values from this rt update.
        tickWasUp = array.get(rtUps_, rtUpdateNo)

        // ————— Update squashed ticks.
        // Figure out which array to update from tick polarity.
        squashedTicks = tickWasUp ? squashedTicksUp_ : squashedTicksDn_
        // Add a tick.
        array.set(squashedTicks, levelToUpdate, array.get(squashedTicks, levelToUpdate) + 1)
        
        // ————— Update squashed volumes.
        // Figure out which array to update from tick polarity.
        squashedVolumes = tickWasUp ? squashedVolumesUp_ : squashedVolumesDn_
        // Add the update's volume.
        array.set(squashedVolumes, levelToUpdate, array.get(squashedVolumes, levelToUpdate) + array.get(rtVolumes_, rtUpdateNo))

        // ————— Update the total of timestamp values for this level.
        squashedTimes = tickWasUp ? squashedTimesUp_ : squashedTimesDn_
        array.set(squashedTimes, levelToUpdate, array.get(squashedTimes, levelToUpdate) + array.get(rtTimes_, rtUpdateNo))
        
        // Go to next rt update.
        rtUpdateNo += 1
// }



// ———————————————————— Calculations and profile drawing {

// ————— Validate chart timeframe when a HTF is used.
if profileResetTfInput != "" and tfInMinutes() > tfInMinutes(profileResetTfInput)
    runtime.error("The chart's timeframe must be >= " + profileResetTfInput)

// ————— Display message if no volume can be found.
if na(volume)
    printMessage("No volume data for this symbol.", GRAY)

// ————— Detect profile reset conditions.
// when a new user-defined TF begins (can be the chart's TF or another).
bool newTf = ta.change(time(profileResetTfInput))
// When Supertrend changes direction.
[_, stDirection] = ta.supertrend(trendSTFactorInput, trendSTLengthInput)
// stChangedDirection = nonRepaintingSecurity(syminfo.tickerid, profileResetTfInput, ta.change(stDirection))
stChangedDirection = ta.change(stDirection)[1]
// When LazyBear's Weis Wave trend changes direction.
// wwChangedDirection = nonRepaintingSecurity(syminfo.tickerid, profileResetTfInput, ta.change(wwLBTrend(close, trendWWSmoothingLengthInput)))
wwChangedDirection = ta.change(wwLBTrend(close, trendWWSmoothingLengthInput))[1]
bool reset = 
  switch
    profileResetOnTF => newTf
    profileResetOnST => stChangedDirection
    profileResetOnWW => wwChangedDirection
    => barstate.isnew

// ————— Only once, init array of strings of lengths 0 to `profileLeftMaxWidthInput` which are used as profile lines.
if barstate.isfirst
    profileInitLevelStrings(profileMaxWidth_)

// —————————— Process realtime updates.
if barstate.isrealtime
    // —————————— 1. Detect when a new profile must be created (as per user's "Reset on" choices) or rescaled because price has breached the previous profile's hi/lo.
    varip scriptJustLoaded = true
    beginNewProfile_ := scriptJustLoaded or (reset and barstate.isnew)
    rescaleProfile_  := high > profileHi_ or low < profileLo_
    bool profileWasRescaled = false
    if beginNewProfile_
        // Script has just been loaded on an open market, or a new TF begins; reset key references.
        profileBarIndex_ := bar_index + profileOffsetInput
        profileHi_ := high
        profileLo_ := low
        qtyOfLevels_ := profileRecalcLevels(profileHi_, profileLo_, profileLevelsMaxQtyInput)
        // Clear rt arrays to begin new profile with a clean set of realtime values.
        array.clear(rtTimes_)
        array.clear(rtPrices_)
        array.clear(rtVolumes_)
        array.clear(rtUps_)
        qtyOfRtUpdates_ := 0
        qtyOfRtVolumes_ := 0.0
        profileWasRescaled := true
    else if rescaleProfile_
        // New hi/lo for which to rescale the profile.
        profileHi_ := math.max(high, profileHi_)
        profileLo_ := math.min(low, profileLo_)
        qtyOfLevels_ := profileRecalcLevels(profileHi_, profileLo_, profileLevelsMaxQtyInput)
        profileWasRescaled := true
    

    // —————————— 2. Fetch and log time, price, volume and polarity of this rt update.
    // Keep track of total volume and ticks for the profile.
    qtyOfRtVolumes_ += getRtUpdate()
    qtyOfRtUpdates_ += 1


    // —————————— 3. Squash rt values into the global scope's `squash*` arrays which contain one up/dn value per profile level.
    squashRtUpdates(profileWasRescaled)
    

    // —————————— 4. Build and draw profiles from user selections.
    // ————— On each update, recreate arrays holding profile display information.
    float[] leftValues   = array.new_float(qtyOfLevels_, 0.0)
    int[]   leftWidths   = array.new_int(qtyOfLevels_,   0)
    int[]   leftHeights  = array.new_int(qtyOfLevels_,   0)
    color[] leftColors   = array.new_color(qtyOfLevels_)
    float[] rightValues  = array.new_float(qtyOfLevels_, 0.0)
    int[]   rightWidths  = array.new_int(qtyOfLevels_,   0)
    int[]   rightHeights = array.new_int(qtyOfLevels_,   0)
    color[] rightColors  = array.new_color(qtyOfLevels_)

    // ————— Remove the previously drawn version of the profile so the new profile doesn't overprint it.
    profileRemovePrevious(profileBarIndex_)

    // ————— Draw new profile(s), provided it's not the first time the script executes on the chart.
    if not scriptJustLoaded
        profileBuildAgeOfLevels(leftHeights, rightHeights)
        
        switch
            profileDOUBLE_VOLUME =>
                profileBuildTwoSides(leftWidths, leftColors, rightWidths, rightColors, VOLUME)
                profileDrawTwoSides(leftWidths, leftHeights, leftColors, rightWidths, rightHeights, rightColors, VOLUME)
            profileDOUBLE_TICKS =>
                profileBuildTwoSides(leftWidths, leftColors, rightWidths, rightColors, TICKS)
                profileDrawTwoSides(leftWidths, leftHeights, leftColors, rightWidths, rightHeights, rightColors, TICKS)
            => 
                switch
                    profileLEFT_VOLUME =>
                        profileBuildOneSide(leftValues, leftWidths, leftColors, LEFT_PROFILE, VOLUME, profileLeftCalcDeltaInput)
                        profileDrawOneSide(leftValues, leftWidths, leftHeights, rightHeights, leftColors, LEFT_PROFILE, VOLUME)
                    profileLEFT_TICKS =>
                        profileBuildOneSide(leftValues, leftWidths, leftColors, LEFT_PROFILE, TICKS, profileLeftCalcDeltaInput)
                        profileDrawOneSide(leftValues, leftWidths, leftHeights, rightHeights, leftColors, LEFT_PROFILE, TICKS)
                switch
                    profileRIGHT_VOLUME =>
                        profileBuildOneSide(rightValues, rightWidths, rightColors, RIGHT_PROFILE, VOLUME, profileRightCalcDeltaInput)
                        profileDrawOneSide(rightValues, rightWidths, leftHeights, rightHeights, rightColors, RIGHT_PROFILE, VOLUME)
                    profileRIGHT_TICKS =>
                        profileBuildOneSide(rightValues, rightWidths, rightColors, RIGHT_PROFILE, TICKS, profileRightCalcDeltaInput)
                        profileDrawOneSide(rightValues, rightWidths, leftHeights, rightHeights, rightColors, RIGHT_PROFILE, TICKS)

    // This tracks when the script has run a first time after being loaded on a chart.
    scriptJustLoaded := false


// ————— Data Window
plotchar(qtyOfRtUpdates_,   "Updates", "", location.top)
plotchar(qtyOfRtVolumes_,   "Volumes", "", location.top)
plotchar(qtyOfLevels_,      "Levels",  "", location.top)
plotchar(na,                "═══════", "", location.top)

// ————— Trend detection markers.
bool stChangedDirectionChanged = ta.change(stChangedDirection)
bool wwChangedDirectionChanged = ta.change(wwChangedDirection)
bool trendChangedUp = (profileResetOnST and stChangedDirection < 0 and stChangedDirectionChanged) or (profileResetOnWW and wwChangedDirection > 0 and wwChangedDirectionChanged)
bool trendChangedDn = (profileResetOnST and stChangedDirection > 0 and stChangedDirectionChanged) or (profileResetOnWW and wwChangedDirection < 0 and wwChangedDirectionChanged)
var bool trendIsUp = na
var bool trendIsDn = na
trendIsUp := not trendIsUp ? trendChangedUp : not trendChangedDn
trendIsDn := not trendIsDn ? trendChangedDn : not trendChangedUp
// Trend change markers.
plotchar(trendMarkerShowChangesInput and (barstate.isrealtime or not trendMarkerChangesRtInput) and trendChangedUp, "Trend change up",   "🠅", location.top, trendMarkerBullColorInput, size = size.tiny)
plotchar(trendMarkerShowChangesInput and (barstate.isrealtime or not trendMarkerChangesRtInput) and trendChangedDn, "Trend change down", "🠇", location.top, trendMarkerBearColorInput, size = size.tiny)
// Trend state markers.
plotchar(trendMarkerShowStatesInput and (barstate.isrealtime or not trendMarkerStatesRtInput) and trendIsUp ? low  : na, "Trend state up",   "🠅", location.absolute, trendMarkerBullColorInput, size = size.tiny)
plotchar(trendMarkerShowStatesInput and (barstate.isrealtime or not trendMarkerStatesRtInput) and trendIsDn ? high : na, "Trend state down", "🠇", location.absolute, trendMarkerBearColorInput, size = size.tiny)

// ————— Candle plots.
bool tickIsUp = array.get(tickUp, 0)
// Body color.
var bool bodyColorModeNO = candleBodyModeInput == CAND0
var bool bodyColorModeUP = candleBodyModeInput == CAND1
var bool bodyColorModeOC = candleBodyModeInput == CAND2
var bool bodyColorModeTR = candleBodyModeInput == CAND3
color bodyColor = switch
    bodyColorModeNO => color(na)
    bodyColorModeOC => tickIsUp     ? candleBodyBullTColorInput : candleBodyBearTColorInput
    bodyColorModeOC => close > open ? candleBodyBullTColorInput : candleBodyBearTColorInput
    bodyColorModeTR => trendIsUp    ? candleBodyBullTColorInput : candleBodyBearTColorInput
// Border color.
var bool borderColorModeNO = candleBorderModeInput == CAND0
var bool borderColorModeUP = candleBorderModeInput == CAND1
var bool borderColorModeOC = candleBorderModeInput == CAND2
var bool borderColorModeTR = candleBorderModeInput == CAND3
color borderColor = switch
    borderColorModeNO => color(na)
    borderColorModeUP => tickIsUp     ? candleBorderBullColorInput : candleBorderBearColorInput
    borderColorModeOC => close > open ? candleBorderBullColorInput : candleBorderBearColorInput
    borderColorModeTR => trendIsUp    ? candleBorderBullColorInput : candleBorderBearColorInput
// Don't color historical bars when required.
bool colorCandles = barstate.isrealtime or not candleColoringRtInput

plotcandle(open, high, low, close, "Candles", colorCandles ? bodyColor : na, wickcolor = colorCandles ? borderColor : na, bordercolor = colorCandles ? borderColor : na)
// }
