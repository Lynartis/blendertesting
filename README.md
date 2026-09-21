readme


//****************
// PROGRAM
//****************
float4 main( const PS_INPUT i ) : SV_TARGET
{
    float MAX_IMPULSE_RADIUS = 350.0f;

    uint   uImpulses         = 0;

    float2 vCurrentMaxOffset = float2( 0.0f, 0.0f );
    float  fCurrentMaxRadius = 0.0f;


    // --------------------------------------------------------
    // TRAILMAP PIXEL -> WORLD XY
    // --------------------------------------------------------

    float2 uv = i.vTex0;
    uv.y = 1.0f - uv.y;

    float2 vWorld =
        WORLD_PIVOT +
        ((uv - float2( 0.5f, 0.5f )) * UV_TO_WORLD);


    // --------------------------------------------------------
    // CURRENT IMPULSES
    // --------------------------------------------------------

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

                // Keep only the CURRENT biggest sphere
                if( fRadius > fCurrentMaxRadius )
                {
                    fCurrentMaxRadius = fRadius;
                    vCurrentMaxOffset = vDir;
                }

                uImpulses++;
            }
        }
    }


    // --------------------------------------------------------
    // PREVIOUS TRAILMAP
    // --------------------------------------------------------

    float4 vPrev =
        texHistory.Sample(
            samplerHistory,
            i.vTex0 - UV_DISPLACEMENT
        );

    float fPrevPower =
        saturate(
            TrailMap_GetPower( vPrev ) -
            DECAY
        );

    float fPrevRadius = 0.0f;

    if( fPrevPower > 0.0f )
    {
        fPrevRadius =
            vPrev.b *
            MAX_IMPULSE_RADIUS;
    }


    // --------------------------------------------------------
    // OUTPUT SELECTION
    //
    // IMPORTANT:
    // We do NOT mix current and previous data.
    // RG + B + A must belong to the same sphere.
    //
    // If previous sphere is bigger, it wins completely.
    // If current sphere is bigger/equal, it wins completely.
    // --------------------------------------------------------

    float2 vRetOffset =
        float2( 0.5f, 0.5f );

    float fRetRadius = 0.0f;
    float fRetPower  = 0.0f;


    // Previous bigger sphere wins completely
    if( fPrevPower > 0.0f &&
        fPrevRadius > fCurrentMaxRadius )
    {
        vRetOffset = vPrev.rg;
        fRetRadius = vPrev.b;
        fRetPower  = fPrevPower;
    }
    // Current sphere wins completely
    else if( uImpulses > 0 )
    {
        vRetOffset =
            saturate(
                (vCurrentMaxOffset / MAX_IMPULSE_RADIUS) *
                0.5f +
                0.5f
            );

        fRetRadius =
            saturate(
                fCurrentMaxRadius /
                MAX_IMPULSE_RADIUS
            );

        fRetPower = 1.0f;
    }
    // Only previous history remains
    else if( fPrevPower > 0.0f )
    {
        vRetOffset = vPrev.rg;
        fRetRadius = vPrev.b;
        fRetPower  = fPrevPower;
    }


    // --------------------------------------------------------
    // FINAL TRAILMAP
    //
    // RG = XY offset from sphere center
    // B  = sphere radius
    // A  = trail power / decay
    // --------------------------------------------------------

    float4 vRet =
        float4(
            0.0f,
            0.0f,
            0.0f,
            0.0f
        );

    vRet.rg = vRetOffset;
    vRet.b  = fRetRadius;
    vRet.a  = fRetPower;

    return vRet;
}
