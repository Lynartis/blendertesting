
//****************
// PROGRAM
//****************
float4 main( const PS_INPUT i ) : SV_TARGET
{
    float MAX_IMPULSE_RADIUS = 350.0f;

    uint   uImpulses   = 0;
    float2 vTotalDir   = float2( 0.0f, 0.0f );
    float  fMaxRadius  = 0.0f;
    float  fDistance   = 0.0f;
    float  fRetPower   = 0.0f;

    float2 uv = i.vTex0;
    uv.y = 1.0f - uv.y;

    float2 vWorld =
        WORLD_PIVOT +
        ((uv - float2( 0.5f, 0.5f )) * UV_TO_WORLD);

    for( uint j = 0; j < MAX_IMPULSES; ++j )
    {
        if( j < (uint)IMPULSES )
        {
            float2 vDir =
                vWorld -
                GET_IMPULSE_POS( j );

            float fSqrDist =
                1.0f -
                saturate(
                    dot( vDir, vDir ) *
                    GET_IMPULSE_RCP_RADIUS( j )
                );

            if( fSqrDist > 0.0f )
            {
                float fRadius =
                    sqrt(
                        1.0f /
                        GET_IMPULSE_RCP_RADIUS( j )
                    );

                // Original style accumulation
                vTotalDir +=
                    vDir *
                    fSqrDist;

                uImpulses++;

                // B now stores radius
                fMaxRadius =
                    max(
                        fMaxRadius,
                        fRadius
                    );
            }
        }
    }

    float4 vPrev =
        texHistory.Sample(
            samplerHistory,
            i.vTex0 - UV_DISPLACEMENT
        );

    float2 vRetDir =
        float2( 0.5f, 0.5f );

    if( uImpulses > 0 )
    {
        // NON-normalized RG:
        // store averaged XY offset vector packed into 0..1
        float2 vAvgDir =
            vTotalDir / uImpulses;

        vRetDir =
            saturate(
                (vAvgDir / MAX_IMPULSE_RADIUS) *
                0.5f +
                0.5f
            );

        // Keep max radius.
        // If you want current-only radius, remove the max with vPrev.b below.
        fDistance =
            saturate(
                max(
                    fMaxRadius,
                    vPrev.b * MAX_IMPULSE_RADIUS
                ) /
                MAX_IMPULSE_RADIUS
            );

        fRetPower = 1.0f;
    }
    else
    {
        // Keep previous packed offset
        vRetDir = vPrev.rg;

        // Keep previous radius
        fDistance = vPrev.b;

        // Original decay behavior
        fRetPower =
            saturate(
                TrailMap_GetPower( vPrev ) -
                DECAY
            );
    }

    float4 vRet =
        float4(
            0.0f,
            0.0f,
            0.0f,
            0.0f
        );

    vRet.rg = vRetDir;
    vRet.b  = fDistance;
    vRet.a  = fRetPower;

    return vRet;
}
