# State Management Rules

## Redux Used Only For:
- Authentication state
- Shared API data
- Cross-page UI state

## Redux NOT Used For:
- Form state
- Modal/dialog temporary state
- Local UI toggles

## Auth Slice Structure (Mandatory)
{
  accessToken,
  refreshToken,
  tokenExpiry,
  isAuthenticated,
  user,
  loading,
  error
}

## All async operations:
- Must use createAsyncThunk
- Must handle loading + error