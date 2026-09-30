// ============================================================================
// TRAIL MAP 3D SEARCH - FIRST VERSION
// ============================================================================
//
// COORDINATE CONVENTION
//
// A = original 3D vegetation/pixel point (worldPos)
// B = vertical projection of A onto trail-map plane Z = 0
// C = first impulse center reconstructed by sampling trail map at B
//
// TrailMap:
//      R,G = normalized direction from sampled pixel -> impulse center
//      B   = normalized distance from sampled pixel -> impulse center
//      A   = impulse power / temporal decay
//
// FIRST IDEA:
//
//      A
//      |
//      |
//      B ---------------- Z = 0
//
// Sample B:
//
//      B -> C
//
// Calculate:
//
//      CA = A - C
//      distanceCA = length(CA)
//
//      CB = normalize(B - C) on the Z=0 plane
//
// First test point:
//
//      B2 = C + CB * distanceCA
//
// Sample B2:
//
//      B2 -> C
//          C is confirmed.
//          A is inside C.
//
//      B2 -> EMPTY
//          C is OUT using this first approximation.
//
//      B2 -> D1
//          D1 may simply be overwriting C.
//          Therefore check opposite side:
//
//          B3 = C - CB * distanceCA
//
//          B3 -> C
//              C confirmed.
//
//          B3 -> EMPTY
//              C out.
//              D1 is queued and tested.
//
//          B3 -> D1
//              C out.
//              D1 queued.
//
//          B3 -> D2
//              C out.
//              D1 and D2 queued.
//
// D1 / D2 use THE EXACT SAME TEST.
//
// Example









//****************
// PROGRAM
//****************
float4 main( const PS_INPUT i ) : SV_TARGET
{
    float MAX_IMPULSE_RADIUS = 350.0f;

    uint uImpulses = 0;

    float2 vTotalDir = float2( 0.0f, 0.0f );
    float fMaxRadius = 0.0f;
    float fRetRadius = 0.0f;
    float fRetPower = 0.0f;


    // --------------------------------------------------------
    // TRAILMAP PIXEL -> WORLD XY
    // --------------------------------------------------------

    float2 uv = i.vTex0;
    uv.y = 1.0f - uv.y;

    float2 vWorld = WORLD_PIVOT + ((uv - float2( 0.5f, 0.5f )) * UV_TO_WORLD);


    // --------------------------------------------------------
    // CURRENT IMPULSES
    // --------------------------------------------------------

    for( uint j = 0; j < MAX_IMPULSES; ++j )
    {
        if( j < (uint)IMPULSES )
        {
            float2 vDir = vWorld - GET_IMPULSE_POS( j );
            float fSqrDist = 1.0f - saturate( dot( vDir, vDir ) * GET_IMPULSE_RCP_RADIUS( j ) );

            if( fSqrDist > 0.0f )
            {
                float fRadius = sqrt( 1.0f / GET_IMPULSE_RCP_RADIUS( j ) );

                vTotalDir += vDir * fSqrDist;
                uImpulses++;
                fMaxRadius = max( fMaxRadius, fRadius );
            }
        }
    }


    // --------------------------------------------------------
    // PREVIOUS TRAILMAP
    // --------------------------------------------------------

    float4 vPrev = texHistory.Sample( samplerHistory, i.vTex0 - UV_DISPLACEMENT );

    float fPrevPowerRaw = TrailMap_GetPower( vPrev );
    float fPrevPower = saturate( fPrevPowerRaw - DECAY );

    float fPrevRadius = 0.0f;

    if( fPrevPowerRaw > 0.0001f )
    {
        float fDecayRatio = fPrevPower / fPrevPowerRaw;
        fPrevRadius = vPrev.b * fDecayRatio;
    }


    // --------------------------------------------------------
    // OUTPUT
    // --------------------------------------------------------

    float2 vRetDir = float2( 0.5f, 0.5f );

    if( uImpulses > 0 )
    {
        // Original structure:
        // current impulse writes current result
        float2 vAvgDir = vTotalDir / uImpulses;
        float fCurrentRadius = saturate( fMaxRadius / MAX_IMPULSE_RADIUS );

        vRetDir = saturate( (vAvgDir / MAX_IMPULSE_RADIUS) * 0.5f + 0.5f );

        // If previous decayed radius is still bigger, keep the whole previous sphere
        // so a smaller current impulse cannot reactivate or cut it.
        if( fPrevPower > 0.0f && fPrevRadius > fCurrentRadius )
        {
            vRetDir = vPrev.rg;
            fRetRadius = fPrevRadius;
            fRetPower = fPrevPower;
        }
        else
        {
            fRetRadius = fCurrentRadius;
            fRetPower = 1.0f;
        }
    }
    else
    {
        // No current impulse: keep previous sphere, but both power and radius decay
        vRetDir = vPrev.rg;
        fRetRadius = fPrevRadius;
        fRetPower = fPrevPower;
    }


    // --------------------------------------------------------
    // FINAL TRAILMAP
    //
    // RG = packed non-normalized XY offset
    // B  = packed radius
    // A  = power / decay
    // --------------------------------------------------------

    float4 vRet = float4( 0.0f, 0.0f, 0.0f, 0.0f );

    vRet.rg = vRetDir;
    vRet.b = fRetRadius;
    vRet.a = fRetPower;

    return vRet;
}





test
//****************
// PROGRAM
//****************
float4 main( const PS_INPUT i ) : SV_TARGET
{
    float MAX_IMPULSE_RADIUS = 350.0f;

    uint uImpulses = 0;

    float2 vTotalDir = float2( 0.0f, 0.0f );
    float fMaxRadius = 0.0f;
    float fRetRadius = 0.0f;
    float fRetPower = 0.0f;


    // --------------------------------------------------------
    // TRAILMAP PIXEL -> WORLD XY
    // --------------------------------------------------------

    float2 uv = i.vTex0;
    uv.y = 1.0f - uv.y;

    float2 vWorld = WORLD_PIVOT + ((uv - float2( 0.5f, 0.5f )) * UV_TO_WORLD);


    // --------------------------------------------------------
    // CURRENT IMPULSES
    // --------------------------------------------------------

    for( uint j = 0; j < MAX_IMPULSES; ++j )
    {
        if( j < (uint)IMPULSES )
        {
            float2 vDir = vWorld - GET_IMPULSE_POS( j );

            float fSqrDist = 1.0f - saturate( dot( vDir, vDir ) * GET_IMPULSE_RCP_RADIUS( j ) );

            if( fSqrDist > 0.0f )
            {
                float fRadius = sqrt( 1.0f / GET_IMPULSE_RCP_RADIUS( j ) );

                vTotalDir += vDir * fSqrDist;

                uImpulses++;

                fMaxRadius = max( fMaxRadius, fRadius );
            }
        }
    }


    // --------------------------------------------------------
    // PREVIOUS TRAILMAP
    // --------------------------------------------------------

    float4 vPrev = texHistory.Sample( samplerHistory, i.vTex0 - UV_DISPLACEMENT );

    float fPrevPower = saturate( TrailMap_GetPower( vPrev ) - DECAY );


    // --------------------------------------------------------
    // OUTPUT
    // --------------------------------------------------------

    float2 vRetDir = float2( 0.5f, 0.5f );

    if( uImpulses > 0 )
    {
        // Keep the original accumulation style, but RG is now
        // NON-normalized so its magnitude is preserved.
        float2 vAvgDir = vTotalDir / uImpulses;

        vRetDir = saturate( (vAvgDir / MAX_IMPULSE_RADIUS) * 0.5f + 0.5f );

        // Current maximum radius packed into B.
        float fCurrentRadius = saturate( fMaxRadius / MAX_IMPULSE_RADIUS );


        // ----------------------------------------------------
        // PREVIOUS BIGGER SPHERE WINS
        //
        // Keep RG + B + A together from history.
        // A smaller current impulse cannot refresh the
        // lifetime of an older larger sphere.
        // ----------------------------------------------------

        if( fPrevPower > 0.0f && vPrev.b > fCurrentRadius )
        {
            vRetDir = vPrev.rg;
            fRetRadius = vPrev.b;
            fRetPower = fPrevPower;
        }
        else
        {
            vRetDir = vRetDir;
            fRetRadius = fCurrentRadius;
            fRetPower = 1.0f;
        }
    }
    else
    {
        vRetDir = vPrev.rg;
        fRetRadius = vPrev.b;
        fRetPower = fPrevPower;
    }


    // --------------------------------------------------------
    // FINAL TRAILMAP
    //
    // RG = packed non-normalized XY offset
    // B  = packed impulse radius
    // A  = power / decay
    // --------------------------------------------------------

    float4 vRet = float4( 0.0f, 0.0f, 0.0f, 0.0f );

    vRet.rg = vRetDir;
    vRet.b = fRetRadius;
    vRet.a = fRetPower;

    return vRet;
}
