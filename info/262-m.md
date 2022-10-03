IllIllI
# Wrong interest will be charged during leap years

## Summary
Leap years have a different number of seconds than normal years

## Vulnerability Detail
The yearly interest is based on an `immutable` number of seconds

## Impact
The wrong interest will be charged for leap years

## Code Snippet
https://github.com/sherlock-audit/2022-08-sentiment-IllIllI000/blob/9ccbe41937173b03f6ea657178eb2efeb3790478/protocol/src/core/DefaultRateModel.sol#L21-L22

## Tool used

Manual Review

## Recommendation
Determine the correct number of seconds in a year, based on which year it is